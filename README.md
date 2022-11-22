# Overview

This simple project demonstrates an issue with the OpenTelemetry Lambda layer
for Python-based Lambda functions.  Specifically, this issue exists in version
1.14.0. The corresponding layer ARN is:

 - `arn:aws:lambda:us-east-1:901920570463:layer:aws-otel-python-amd64-ver-1-14-0:1`

# Issue Details

Lambda function handlers can be specified in at least the two following ways for
Python functions:

 - `package1.package2.module.handler`
 - `package1/package2/module.handler`

Either of these forms is valid and executes `module.handler` as expected when
the function is invoked. However, the OpenTelemetry Lambda layer does not
properly handle the second form. If it is enabled, invocation results in the
following error response when slash-based paths are used:

```json
{
  "errorMessage": "Unable to import module 'otel_wrapper': No module named 'package1/package2/module'",
  "errorType": "Runtime.ImportModuleError",
  "requestId": "b8fefc59-1377-4cf3-aec6-ba7ea8cdfa84",
  "stackTrace": []
}
```

The
[otel_wrapper.py](https://github.com/open-telemetry/opentelemetry-lambda/blob/f48ed2cff0c18722639d6062b93b02c64fc9cec4/python/src/otel/otel_sdk/otel_wrapper.py)
script handles both forms by modifying the `ORIG_HANDLER` path (see
[here](https://github.com/open-telemetry/opentelemetry-lambda/blob/f48ed2cff0c18722639d6062b93b02c64fc9cec4/python/src/otel/otel_sdk/otel_wrapper.py#L43)),
but
[AwsLambdaInstrumentor().instrument()](https://github.com/open-telemetry/opentelemetry-lambda/blob/f48ed2cff0c18722639d6062b93b02c64fc9cec4/python/src/otel/otel_sdk/otel_wrapper.py#L52)
does not and instrumenting the function fails with the error above.

I believe the error originates within the call to `wrapt.wrap_function_wrapper`
[here](https://github.com/open-telemetry/opentelemetry-python-contrib/blob/b6b269064c5e9bf1304dcdbe6686cca2dd87eb42/instrumentation/opentelemetry-instrumentation-aws-lambda/src/opentelemetry/instrumentation/aws_lambda/__init__.py#L347)
when patching occurs because the slash-based path format is not a valid Python
import path.

# Reproducing the Issue
This repository contains some simple source code and CloudFormation templates to
reproduce the issue.  To reproduce the issue, you'll just need to deploy the
stack and invoke the functions it creates.

There are four functions which will be created:

 - SlashPathWithLayerBroken <-- **BROKEN**
 - SlashPathNoLayerWorks
 - DotPathNoLayerWorks
 - DotPathWithLayerWorks

## Deploying the Stack
Just run the following commands (replacing text in <> with your own values):

```bash
aws cloudformation package --template-file stack.yaml --s3-bucket <your-bucket> --profile <your-profile> > deploy-template.yaml
aws cloudformation deploy --region us-east-1 --template-file deploy-template.yaml --stack-name otel-issue-demo --profile personal --capabilities CAPABILITY_IAM
```

> **Note**:
> 
> If you deploy in a region other than `us-east-1`, you'll also need to specify the
> `OpenTelemetryLayer` parameter with the same region specified in the layer ARN.

# Resolving the Issue

Possible resolutions and workarounds are described below.

## Workaround
This issue can easily be worked around by specifying the handler location using
dot-based paths. Nothing about project structure needs to change.

## Modify ORIG_HANDLER in `otel_wrapper.py`
Modifying the `ORIG_HANDLER` environment variable within `otel_wrapper.py`
before instrumentation occurs by replacing it with the modified path that
is computed for use within itself resolves the issue.

While this works, it does mutate the `ORIG_HANDLER` variable in a way which
makes it no longer truly the original handler path. Side effects outside of the
intended scope are definitely possible, but it might also help prevent similar
issues sharing ths root cause.

This is the fix I implemented when debugging this issue. It does work and requires
only minimal modifications.

## Modify `AwsLambdaInstrumentor().instrument()`
The `AwsLambdaInstrumentor().instrument()` method could be modified to compute
the path to the `ORIG_HANDLER` it loads by replacing the slashes with dots as
the `otel_wrapper.py` script does. This would ensure that the path is valid for
import, preventing the error.

This fix duplicates the path modification logic, but it reduces the potential
of unexpected side effects by keeping the scope small and targeted.

