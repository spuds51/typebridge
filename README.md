# TypeBridge

Typescript toolbox for AWS EventBridge

## Advantages

- Programmatical definition of your application events
- Typed publish and consume APIs
- Automatically batches `putEvents` call when publishing more than 10 events at a time
- Check for event payload size before publishing

## Quick install

### Add typebridge dependency

`npm i typebridge --save`

### Define your bus and events

```ts
import { EventBridge } from 'aws-sdk';
import { Bus, Event } from 'typebridge';

const MyBus = new Bus({
  name: 'applicationBus',
  EventBridge: new EventBridge(),
});

export const MyEventPayloadSchema = {
  type: 'object',
  properties: {
    stringAttribute: { type: 'string' };
    numberAttribute: { type: 'integer' };
  }
  required: ['stringAttribute'],
  additionalProperties: false
} as const;

export const MyEvent = new Event({
  name: 'MyEvent',
  bus: SafetrackerBus,
  schema: MyEventPayloadSchema
});
```

### Use the Event class to publish

```ts
import { MyEvent } from './events.ts';

export const handler = async (event) => {
  await MyEvent.publish({
    stringAttribute: 'string',
    numberAttribute: 12,
  })

  return 'Event published !'
};
```

Typechecking is automatically enabled:

```ts
  await MyEvent.publish({
    stringAttribute: 'string',
    numberAttribute: 12,
    // the following line will trigger a Typescript error
    anotherAttribute: 'wrong'
  })
```

### Use the Event class to generate trigger rules

Using the serverless framework with `serverless.ts` service file:


```ts
import type { Serverless } from 'serverless/aws';

const serverlessConfiguration: Serverless = {
  service: 'typebridge-test',
  provider: {
    name: 'aws',
    runtime: 'nodejs12.x',
  },
  functions: {
    hello: {
      handler: 'MyEventHandler.handler',
      events: [
        {
          eventBridge: {
            eventBus: 'applicationBus',
            pattern: NewUserConnectedEvent.computePattern(),
          },
        },
      ],
    }
  }
}

module.exports = serverlessConfiguration;
```

### Use the Event class to type input event

Using the serverless framework with `serverless.ts` service file:


```ts
import { EventBridgeEvent } from 'aws-lambda/trigger/eventbridge';
import { FromSchema } from 'json-schema-to-ts';
import { MyEvent, MyEventPayload } from './events.ts';

export const handler = (event: EventBridgeEvent<typeof MyEvent.name, FromSchema<typeof MyEventPayloadSchema>>) => {
  // Typed as string
  return event.detail.stringAttribute;
}
```

### Use the Event class to validate the input event

Using [middy](https://github.com/middyjs/middy) middleware stack in your lambda's handler, you can throw an error before your handler's code being executed if the input event `detail` property does not satisfy the JSON-schema used in `MyEvent` constructor.

```ts
import { MyEvent } from './events.ts';

const handler = (event) => {
  return 'Validation succeeded';
}

// If event.detail does not match the JSON-schema supplied to MyEvent constructor, the middleware will throw an error
export const main = middy(handler).use(MyEvent.validationMiddleware())
```
