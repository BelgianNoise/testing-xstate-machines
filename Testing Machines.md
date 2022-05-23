# Testing Machines

## This file structure will be used throughout the whole document
 - example.context.ts
 - example.events.ts
 - example.machine.ts
 - example.services.ts
 - example.states.ts

## Goal

The goal of this document is to provide a good base for understanding how we test machines and their affiliated files. All tests provided should result in 100% test coverage for all files except `example.services.ts`.

## Example files

The syntax of the example files below might not be 100% correct.

### example.context.ts

<details>
  <summary>Show file</summary>

  ```
  export interface ExampleContext {
    someValue: string;
  }

  ```

</details>

### example.events.ts

<details>
  <summary>Show file</summary>

```
import { EventObject } from 'xstate';

export enum ExampleEvents {
  CLICKED_GO_TO_FIRST_PAGE = '[ExampleEvents: Clicked Go To First Page]',
  CLICKED_GO_TO_SECOND_PAGE = '[ExampleEvents: Clicked Go To Second Page]',
}

export class ClickedGoToFirstPageEvent implements EventObject {

  public type: ExampleEvents.CLICKED_GO_TO_FIRST_PAGE = ExampleEvents.CLICKED_GO_TO_FIRST_PAGE;

}

export class ClickedGoToSecondPageEvent implements EventObject {

  public type: ExampleEvents.CLICKED_GO_TO_SECOND_PAGE = ExampleEvents.CLICKED_GO_TO_SECOND_PAGE;

}


export type ExampleEvent =
  | ClickedGoToFirstPageEvent
  | ClickedGoToSecondPageEvent;

```

</details>

### example.machine.ts

This is a simple machine that uses 3 states, 2 for indicating that the user is looking at a page and 1 to indicate that some data is being loaded in the background. A user can navigate between the 2 pages. When transitioning to the second page, the data will be loaded in the background.

<details>
  <summary>Show file</summary>

```
import { MachineConfig, StateMachine } from 'xstate';
import { ExampleContext } from './example.context';
import { ExampleState, ExampleStates, ExampleStateSchema } from './example.states';
import { ExampleEvent, ExampleEvents } from './example.events';
import { loadExample } from './example.services';

// eslint-disable-next-line max-len
export type ExampleMachine = StateMachine<ExampleContext, ExampleStateSchema, ExampleEvent, ExampleState>;

export const exampleMachine: MachineConfig<ExampleContext, ExampleStateSchema, ExampleEvent> = {

  initial: ExampleStates.VIEWING_FIRST_PAGE,

  states: {
    [ExampleStates.VIEWING_FIRST_PAGE]: {
      on: {
        [ExampleEvents.CLICKED_GO_TO_SECOND_PAGE]: ExampleStates.LOADING_EXAMPLE_DATA,
      },
    },
    [ExampleStates.LOADING_EXAMPLE_DATA]: {
      invoke: {
        src: (c: ExampleContext, e: ExampleEvent) => loadExample(c, e),
        onDone: {
          actions: assign({ someValue: (c: ExampleContext, ev: DoneInvokeEvent<string>) => ev.data }),
          target: ExampleStates.VIEWING_SECOND_PAGE,
        },
        onError: {
          actions: log('An error occurred loading example data'),
          target: ExampleStates.VIEWING_FIRST_PAGE,
        },
      }
    },
    [ExampleStates.VIEWING_SECOND_PAGE]: {
      on: {
        [ExampleEvents.CLICKED_GO_TO_FIRST_PAGE]: ExampleStates.VIEWING_FIRST_PAGE,
      },
    },
  },

};

```

</details>

### example.services.ts

<details>
  <summary>Show file</summary>

```
import { ExampleContext } from './example.context';
import { ExampleEvent } from './example.events';

export const loadExample = async (
  context: ExampleContext,
  ev: ExampleEvent,
): Promise<string> => {
  return 'someValue';
}
```

</details>

### example.states.ts

<details>
  <summary>Show file</summary>

```
import { StateSchema } from 'xstate';
import { ExampleContext } from './example.context';

export enum ExampleStates {
  VIEWING_FIRST_PAGE = '[ExampleStates: Viewing First Page]',
  VIEWING_SECOND_PAGE = '[ExampleStates: Viewing Second Page]',
  LOADING_EXAMPLE_DATA = '[ExampleStates: Loading Example Data]',
}

export interface ExampleState {
  value:
  | ExampleStates.VIEWING_FIRST_PAGE
  | ExampleStates.VIEWING_SECOND_PAGE
  | ExampleStates.LOADING_EXAMPLE_DATA;
  context: ExampleContext;
}

export interface ExampleStateSchema extends StateSchema<ExampleContext> {
  states: {
    [key in ExampleStates]?: StateSchema<ExampleContext>;
  };
}

```

</details>

## Testing

### example.machine.spec.ts

#### Where to start

Don't immediately start coding. First look at the code of your machine and think about which tests you need to cover every possible path. Write out every scenario in an `it` so you don't forget.

For this machine we will want to test:
 - Does the machine transition to its initial state correctly
 - When in VIEWING_FIRST_PAGE, a CLICKED_GO_TO_SECOND_PAGE_EVENT should transition the machine to LOADING_EXAMPLE_DATA
 - When in VIEWING_SECOND_PAGE, a CLICKED_GO_TO_FIRST_PAGE_EVENT should transition the machine to VIEWING_FIRST_PAGE
 - When the machine reaches LOADING_EXAMPLE_DATA
   - Check if the correct service function is called, loadExample in this case
   - Check if when the invoke resolves, the machine transitions to VIEWING_SECOND_PAGE and it sets the correct value to the context (see assign in onDone)
   - Check if when the invoke rejects (an error is thrown inside the service function), the machine transitions to VIEWING_FIRST_PAGE

<details>
  <summary>This list translates into:</summary>

```
describe('exampleMachine', () => {

  let machine: InterpreterFrom<ExampleMachine>;

  beforeEach(() => {

    jest.resetAllMocks();

    machine = interpret(createMachine<ExampleContext, ExampleEvent, ExampleState>(exampleMachine).withContext({ someValue: 'blabla' }));

    // mock the service functions as these should be tested in their own file and may not be covered by this file
    jest.spyOn(services, 'loadExample').mockResolvedValue('blablaReturnedByServiceFunction');

  });

  afterEach(() => {

    machine.stop();

  });

  it('should instantiate', () => {

    machine.start();

    expect(machine).toBeTruthy();

  });

  it(`should transition to ${ExampleStates.VIEWING_FIRST_PAGE} as its initial state`, async () => {

  });

  describe(`${ExampleStates.VIEWING_FIRST_PAGE}`, () => {

    it(`should transition to ${ExampleStates.LOADING_EXAMPLE_DATA} on ${ExampleEvents.CLICKED_GO_TO_SECOND_PAGE_EVENT}`, async () => {

    });

  });

  describe(`${ExampleStates.VIEWING_SECOND_PAGE}`, () => {

    it(`should transition to ${ExampleStates.VIEWING_FIRST_PAGE} on ${ExampleEvents.CLICKED_GO_TO_FIRST_PAGE_EVENT}`, async () => {

    });

  });

  describe(`${ExampleStates.LOADING_EXAMPLE_DATA}`, () => {

    describe('INVOKE', () => {

      it('should call example.services.loadExample', async () => {

      });

      it(`should transition to ${ExampleStates.VIEWING_SECOND_PAGE} when invoke resolves`, async () => {

      });

      it(`should transition to ${ExampleStates.VIEWING_FIRST_PAGE} when invoke rejects`, async () => {

      });

    });

  });

});
```


</details>

#### The complete file (for reference)

<details> 
  <summary>Show file</summary>
  
  ```
  import { createMachine, interpret, InterpreterFrom } from 'xstate';
  import { exampleMachine, ExampleMachine } from './example.machine';
  import { ClickedGoToFirstPageEvent, ClickedGoToSecondPageEvent, ExampleEvent, ExampleEvents } from './example.events';
  import { ExampleState, ExampleStates } from './example.states';
  import { ExampleContext } from './example.context';
  import * as services from './example.services';

  describe('exampleMachine', () => {

    let machine: InterpreterFrom<ExampleMachine>;

    beforeEach(() => {

      jest.resetAllMocks();

      machine = interpret(createMachine<ExampleContext, ExampleEvent, ExampleState>(exampleMachine).withContext({ someValue: 'blabla' }));

      // mock the service functions as these should be tested in their own file and may not be covered by this file
      jest.spyOn(services, 'loadExample').mockResolvedValue('blablaReturnedByServiceFunction');

    });

    afterEach(() => {

      machine.stop();

    });

    it('should instantiate', () => {

      machine.start();

      expect(machine).toBeTruthy();

    });

    it(`should transition to ${ExampleStates.VIEWING_FIRST_PAGE} as its initial state`, async () => {

      const onTransitionCheck = new Promise<void>((resolve, reject) => {

        machine.onTransition((state) => {

          if (state.matches(ExampleStates.VIEWING_FIRST_PAGE)) return resolve();
          reject();

        });

      });

      machine.start();
      // No extra events should be sent to the machine as we are trying to test its initial state

      await expect(onTransitionCheck).resolves.toBeUndefined();

    });

    describe(`${ExampleStates.VIEWING_FIRST_PAGE}`, () => {

      it(`should transition to ${ExampleStates.LOADING_EXAMPLE_DATA} on ${ExampleEvents.CLICKED_GO_TO_SECOND_PAGE_EVENT}`, async () => {

        const onTransitionCheck = new Promise<void>((resolve, reject) => {

          machine.onTransition((state) => {

            if (state.matches(ExampleStates.VIEWING_FIRST_PAGE)) return;
            if (state.matches(ExampleStates.LOADING_EXAMPLE_DATA)) return resolve();
            reject();

          });

        });

        machine.start();
        machine.send(new ClickedGoToSecondPageEvent());

        await expect(onTransitionCheck).resolves.toBeUndefined();

      });

    });

    describe(`${ExampleStates.VIEWING_SECOND_PAGE}`, () => {

      it(`should transition to ${ExampleStates.VIEWING_FIRST_PAGE} on ${ExampleEvents.CLICKED_GO_TO_FIRST_PAGE_EVENT}`, async () => {

        const onTransitionCheck = new Promise<void>((resolve, reject) => {

          let initialFirstPagePassed = false;

          machine.onTransition((state) => {

            if (state.matches(ExampleStates.VIEWING_FIRST_PAGE) && initialFirstPagePassed) return resolve();
            if (state.matches(ExampleStates.VIEWING_FIRST_PAGE)) return initialFirstPagePassed = true;
            if (state.matches(ExampleStates.LOADING_EXAMPLE_DATA)) return;
            if (state.matches(ExampleStates.VIEWING_SECOND_PAGE)) return;
            reject();

          });

        });

        machine.start();
        // First go to the second page
        machine.send(new ClickedGoToSecondPageEvent());
        // Return to the first page
        machine.send(new ClickedGoToFirstPageEvent());

        await expect(onTransitionCheck).resolves.toBeUndefined();

      });

    });

    describe(`${ExampleStates.LOADING_EXAMPLE_DATA}`, () => {

      describe('INVOKE', () => {

        it('should call example.services.loadExample', async () => {

          // Make sure we make it to INVOKE
          const onTransitionCheck = new Promise<void>((resolve, reject) => {

            machine.onTransition((state) => {

              if (state.matches(ExampleStates.VIEWING_FIRST_PAGE)) return;
              if (state.matches(ExampleStates.LOADING_EXAMPLE_DATA)) return resolve();
              reject();

            });

          });

          machine.start();
          machine.send(new ClickedGoToSecondPageEvent());

          await expect(onTransitionCheck).resolves.toBeUndefined();
          expect(services.loadExample).toHaveBeenCalledTimes(1);

        });

        it(`should transition to ${ExampleStates.VIEWING_SECOND_PAGE} and set someValue in the context when invoke resolves`, async () => {

          const onTransitionCheck = new Promise<void>((resolve, reject) => {

            machine.onTransition((state) => {

              if (state.matches(ExampleStates.VIEWING_FIRST_PAGE)) return;
              if (state.matches(ExampleStates.LOADING_EXAMPLE_DATA)) return ;
              if (state.matches(ExampleStates.VIEWING_SECOND_PAGE)) return resolve();
              reject();

            });

          });

          const onChangeCheck = new Promise<void>((resolve) => {

            machine.onChange((context: AppContext) => {

              if (context.someValue === 'blablaReturnedByServiceFunction') return resolve();

            });

          });

          machine.start();
          machine.send(new ClickedGoToSecondPageEvent());

          await expect(onTransitionCheck).resolves.toBeUndefined();
          await expect(onChangeCheck).resolves.toBeUndefined();

        });

        it(`should transition to ${ExampleStates.VIEWING_FIRST_PAGE} when invoke rejects`, async () => {

          jest.spyOn(services, 'loadExample').mockRejectedValueOnce(new Error());

          const onTransitionCheck = new Promise<void>((resolve, reject) => {

            let initialFirstPagePassed = false;

            machine.onTransition((state) => {

              if (state.matches(ExampleStates.VIEWING_FIRST_PAGE) && initialFirstPagePassed) return resolve();
              if (state.matches(ExampleStates.VIEWING_FIRST_PAGE)) return initialFirstPagePassed = true;
              if (state.matches(ExampleStates.LOADING_EXAMPLE_DATA)) return;
              reject();

            });

          });

          machine.start();
          machine.send(new ClickedGoToSecondPageEvent());

          await expect(onTransitionCheck).resolves.toBeUndefined();

        });

      });

    });

  });

  ```

</details>

#### explanation / how does the test work

<details>
  <summary>Let's take a look at the longest test:</summary>

```
  it(`should transition to ${ExampleStates.VIEWING_FIRST_PAGE} when invoke rejects`, async () => {

    jest.spyOn(services, 'loadExample').mockRejectedValueOnce(new Error());

    const onTransitionCheck = new Promise<void>((resolve, reject) => {

      let initialFirstPagePassed = false;

      machine.onTransition((state) => {

        if (state.matches(ExampleStates.VIEWING_FIRST_PAGE) && initialFirstPagePassed) return resolve();
        if (state.matches(ExampleStates.VIEWING_FIRST_PAGE)) return initialFirstPagePassed = true;
        if (state.matches(ExampleStates.LOADING_EXAMPLE_DATA)) return;
        reject();

      });

    });

    machine.start();
    machine.send(new ClickedGoToSecondPageEvent());

    await expect(onTransitionCheck).resolves.toBeUndefined();

  });
```


</details>

In this test we expect the machine to make these transitions:
 1. VIEWING_FIRST_PAGE
 2. LOADING_EXAMPLE_DATA
 3. VIEWING_FIRST_PAGE

The most important part of machine tests will always be the `onTransitionCheck`.

We create a new Promise that will resolve() when all goes according to plan and reject() when something happens that we do not expect to happen, most tests will have the same result when we leave out the reject() and will time out instead of fail. But it can happen that a test succeeds when we leave out the reject(). So most important is that we only allow the machine to follow the path we want / expect it to follow.


Every time the machine transitions it will go though:

```
  if (state.matches(ExampleStates.VIEWING_FIRST_PAGE) && initialFirstPagePassed) return resolve();
  if (state.matches(ExampleStates.VIEWING_FIRST_PAGE)) return initialFirstPagePassed = true;
  if (state.matches(ExampleStates.LOADING_EXAMPLE_DATA)) return;
  reject();
```

NOTICE:
- every line should include `return`, otherwise it will continue going through all following if statement and eventually also reject().
- This test also contains a great example for when you want to test if the machine transitions to a state multiple times.
- This test contains an example of how you make an `invoke` reject to end up in its `onError`.


#### Testing changes in the state

<details>
  <summary>Test:</summary>

```
  it(`should transition to ${ExampleStates.VIEWING_SECOND_PAGE} and set someValue in the context when invoke resolves`, async () => {

    const onTransitionCheck = new Promise<void>((resolve, reject) => {

      machine.onTransition((state) => {

        if (state.matches(ExampleStates.VIEWING_FIRST_PAGE)) return;
        if (state.matches(ExampleStates.LOADING_EXAMPLE_DATA)) return ;
        if (state.matches(ExampleStates.VIEWING_SECOND_PAGE)) return resolve();
        reject();

      });

    });

    const onChangeCheck = new Promise<void>((resolve) => {

      machine.onChange((context: AppContext) => {

        if (context.someValue === 'blablaReturnedByServiceFunction') return resolve();

      });

    });

    machine.start();
    machine.send(new ClickedGoToSecondPageEvent());

    await expect(onTransitionCheck).resolves.toBeUndefined();
    await expect(onChangeCheck).resolves.toBeUndefined();

  });
```

</details>

In this test we are checking if the context is being changed to what we expect. when LOADING_EXAMPLE_DATA resolves it should set someValue in the context. Here we check exactly that, context.someValue should be equal to the result of the service function loadExample (look at the mocked response in beforeEach() ).
