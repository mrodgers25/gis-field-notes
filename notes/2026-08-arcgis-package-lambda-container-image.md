# The arcgis Python package is too big for a Lambda zip

*Worked on: August 2026*

I wanted a nightly Lambda to write CRM changes into a hosted feature layer using the ArcGIS API for Python. The package wouldn't fit in a Lambda zip, so I moved the ArcGIS write step into a container image and split the pipeline into two chained Lambdas.

## The situation

The nightly sync reads from a donor CRM and writes to ArcGIS Online. My first plan was one Lambda, deployed as a zip with its dependencies bundled in. The `arcgis` package and its dependencies are large, and the bundle went past Lambda's size limits for a zip deployment.

I could have tried trimming dependencies or using layers. I didn't want to spend my time fighting package size every time something upgraded, so I went another direction.

## What I did

**Split the work into two Lambdas.**

1. **Build-delta Lambda (zip).** Talks to the CRM, works out what changed, and writes a delta file to S3. It only needs light dependencies, so a zip works fine.
2. **Write Lambda (container image).** Reads the delta file and applies it to the hosted layer with the `arcgis` package. This one is built as a container image and stored in ECR. Container images allow a much larger package than a zip, so the heavy dependency isn't a problem.

**Chain them asynchronously through S3.** The first Lambda doesn't call the second one directly and wait for it. It writes the delta to S3, and that file drops the pipeline into the next step. Nothing sits waiting on the other function, and each step can be run, logged, and debugged on its own.

```dockerfile
FROM public.ecr.aws/lambda/python:3.12

COPY requirements.txt .
RUN pip install -r requirements.txt --target "${LAMBDA_TASK_ROOT}"

COPY app.py ${LAMBDA_TASK_ROOT}
CMD ["app.handler"]
```

The handler itself is ordinary: read the delta from S3, connect to the portal, and apply edits.

```python
def handler(event, context):
    delta = load_delta_from_s3(event)
    gis = connect_to_portal()
    layer = get_target_layer(gis)
    apply_updates(layer, delta)
```

## What was tricky

- Container images need a build and push step to ECR before the Lambda can use them, which is more ceremony than uploading a zip. I treat the image as the deployable unit and rebuild it deliberately.
- Splitting the pipeline means the file in S3 is the contract between the two steps. It has to be a clean, self-describing delta, because the write step trusts it.
- Because the steps are chained through a file, it's easy to be tempted to trigger the second one by hand. That's a risk worth designing around, since the write step should only ever run on a freshly built delta.

## The upside I didn't plan for

The split turned out to be a better design on its own merits. The CRM-facing code and the ArcGIS-facing code are separate, so I can test the delta logic without touching the layer, and a failure in one step doesn't hide inside the other.

## Takeaway

If the `arcgis` package won't fit in a Lambda zip, don't fight it. Put the ArcGIS step in a container image in ECR, keep the other steps as lightweight zips, and pass work between them with a file in S3. You end up with smaller, simpler functions and a clear boundary between them.
