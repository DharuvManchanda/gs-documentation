AWS as a datasource plugin: Turbocharge your app by tapping into Amazon Web Services. Unleash the power of cloud-based data, storage, and more to supercharge your application. 🚀 

## Steps to use aws plug-in in godspeed framework:

### How to install
- Create a godspeed project from the CLI , open the created project in vscode and then add the plugin from the CLI of vscode, select the `@godspeedsystems/plugins-aws-as-datasource` to integrate the plugin.

```
> godspeed plugin add
       ,_,   ╔════════════════════════════════════╗
      (o,o)  ║        Welcome to Godspeed         ║
     ({___}) ║    World's First Meta Framework    ║
       " "   ╚════════════════════════════════════╝
? Please select godspeed plugin to install: (Press <space> to select, <Up and Down> to move rows)
┌──────┬────────────────────────────────────┬────────────────────────────────────────────────────────────────────┐
│      │ Name                               │ Description                                                        │
├──────┼────────────────────────────────────┼────────────────────────────────────────────────────────────────────┤
│  ◯   │ prisma-as-datastore                │ Prisma as a datasource plugin for Godspeed Framework.              │
├──────┼────────────────────────────────────┼────────────────────────────────────────────────────────────────────┤
│ ❯◯   │ aws-as-datasource                  │ aws as datasource plugin for Godspeed Framework                    │
└──────┴────────────────────────────────────┴────────────────────────────────────────────────────────────────────┘
```
- The plugin can also be directly installed by running `npm i @godspeedsystems/aws-as-datasource` command

### Configuration

In your <aws_ds_name>.yaml file, you will need to configure
- type: aws (type of the datasource)
- default_client_config (optional) for initializing your clients, as per the aws config specs
- Client type to client name mappings via the `types` key
- `services` contains settings for the services you want to invoke via this datasource. 
  - Each service has a type like s3, lamdba etc.
  - They can have their own config overriding the default under the `config` key
  - Note: There can be multiple services configured for the same type. Check `s3` and `s3_1` below

```yaml
type: aws
default_client_config: #any aws specific configurations
  credentials:
    accessKeyId: <%config.accessKeyId%>
    secretAccessKey: <%config.secretAccessKey%>
# service type is the name of the npm module for ex. @aws-sqk/client-dynamodb or @aws-sqk/client-s3 etc
# The `types` key can have service type to sdk's client names mappings when coding
types: #mappings
  dynamodb: DynamoDB
  s3: S3
  lambda: Lambda
  ssm: SSM
  sqs: SQS
services:
  s3:
    type: s3
    config:
      region: <%config.anotherAccessKeyId%>
      credentials:
        accessKeyId: <%config.anotherAccessKeyId%>
        secretAccessKey: <%config.anotherSecretAccessKey%>
  s3_1: #uses default config
    type: s3
  dynamodb:
    type: dynamodb
  sqs:
    type: sqs
  ssm:
    type: ssm
  lamdba:
    type: lambda
```

### Example usage

In an event, we establish HTTP endpoint that accepts json objects in request body. When this endpoint is invoked, it triggers the `aws_list` function with the args coming from request body.

#### Example event schema
```yaml
# event for create

"http.post./aws":
  fn: aws_list
  body:
    type: object
  responses:
    200:
      content:
         application/json:

```

#### Example YAML workflow

In workflow we need to mention `datasource.aws.${serviceName}.${method}` as function (fn) to perform operations in this case `datasource.aws.s3.listObjects`.

```yaml
id: aws_workflow
tasks:
  - id: aws_list
    fn: datasource.aws.s3.listObjects
    args: <% inputs.body %>
```
#### Example TS workflow
```ts
import { GSContext, GSDataSource, GSStatus } from "@godspeedsystems/core";

export default async function (ctx: GSContext, args: any) {
    const ds: GSDataSource = ctx.datasources.aws;
    const response = await ds.execute(ctx, {
         //Pass exactly same args as this aws service's method takes
        ...args,
        //Along with args, pass meta object
        // meta can contain {entityName, method}
        meta: {entityName: 's3', method: 'listBuckets'},
        //Or meta can contain {fnNameInWorkflow} which is same as 
        //the 'fn' that we write when invoking datasource from yaml workflow
        //For example, this will also work
        //meta: {fnNameInWorkflow: 'datasource.aws.s3.listBuckets'}
    });
    return response;
}
```
### Example to upload file on aws s3

Event
```yaml
# event for upload s3 file

"http.post./aws":
  fn: aws_upload
  body:
  content:
    multipart/form-data:
        schema:
        $ref: '#/definitions/event/upload_file'
  responses:
    200:
      content:
         application/json:

```
the above event calling `aws_upload.ts` workflow from `src/functions/` 
 
```js 
import { GSContext, GSStatus } from "@godspeedsystems/core";
import fs from 'fs'

module.exports = async (ctx: GSContext) => {
  const { files: { myfile } } = ctx.inputs.data;
  const { datasources, logger } = ctx;
  try {
    return new Promise((resolve, reject) => {
      fs.readFile(myfile.tempFilePath, async function (err, data) {
        if (err) {
        resolve(new GSStatus(false, 500, 'S3 document upload failed', { error: { message: err.message } }));

        } // Something went wrong!
        const contentType = ctx.inputs.data.headers['content-type']
        var params = {
          Key: myfile.name, //file name
          Body: data,
          Bucket: 'userdocs123',
          ContentType: contentType,
        };

        const res = await datasources.aws.client.s3.putObject(params);

        resolve(new GSStatus(true, 200, 'successfully uploaded document', res, undefined))

      })
    });

  } catch (e: any) {
    logger.error('S3 document upload failed %o', e)
    return new GSStatus(false, 500, 'S3 document upload failed', { error: { message: e.message } })
  }

};
```
