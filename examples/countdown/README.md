# Countdown

This is an example of using redux-logic to govern the logic for a countdown timer.

The timer will start when the action type of `TIMER_START` is received. While it is running it dispatches a `TIMER_DECREMENT` every second until it reaches 0.

It will countdown each second until reaching 0 when it will dispatch a `TIMER_END`.

An action type of `TIMER_CANCEL` will stop the timer and an action type of `TIMER_RESET` will both stop the timer and reset the count to the initial value of 10.

If you try to start the timer when it is already running, the timerStartLogic will detect this and drop the action so it doesn't continue to the reducers.

If you try to start the timer when the value is already zero, the timerStartLogic will detect this as an error condition and will send a TIMER_START_ERROR action to update the status that you should first reset the counter.

It builds action creators and reducers without using any helper libraries.

It showcases some of the declarative functionality built into redux-logic, so simply by specifying a cancelType, we enable this code to be cancellable. No code had to be written by us to leverage that functionality. Just be declaring the `cancelType`, when a cancellation action is received future dispatching is disabled, however to be a good citizen we should cleanup any resourses that we created, in this case we should stop the timer we started. We can listen to the cancelled$ observable and if it fires we can perform our cleanup.

Finally this also shows how to use dispatch for a long running task with multiple dispatches. To perform multiple dispatches, pass `{ allowMore: true }` for the second argument `options`. Alternatively you can simply dispatch an observable. See [Advanced usage in the API docs](../../docs/api.md#advanced-usage)


```js
// in src/timer/logic.js

const timerStartLogic = createLogic({
  type: TIMER_START,
  cancelType: [TIMER_CANCEL, TIMER_RESET, TIMER_END], // any will cancel

  // check to see if it is valid to start, > 0
  validate({ getState, action }, allow, reject) {
    const state = getState();
    if (timerSel.status(state) !== 'stopped') {
      // already started just silently reject
      return reject();
    }
    if (timerSel.value(state) > 0) {
      allow(action);
    } else {
      reject(timerStartError(new Error('can\'t start, already zero')));
    }
  },

  process({ cancelled$ }, dispatch) {
    const interval = setInterval(() => {
      // passing allowMore: true option to keep open for more dispatches
      dispatch(timerDecrement(), { allowMore: true });
    }, 1000);

    // The declarative cancellation already stops future dispatches
    // but we should go ahead and stop the timer we created.
    // If cancelled, stop the time interval
    cancelled$.subscribe(() => {
      clearInterval(interval);
      dispatch(); // dispatch nothing to tell logic we are done
    });
  }
});

const timerDecrementLogic = createLogic({
  type: TIMER_DECREMENT,

  validate({ getState, action }, allow, reject) {
    const state = getState();
    if (timerSel.value(state) > 0) {
      allow(action);
    } else { // shouldn't get here, but if does end
      reject(timerEnd());
    }
  },

  process({ getState }, dispatch) {
    // unless other middleware/logic introduces async behavior, the
    // state will have been updated by the reducers by now
    const state = getState();
    if (timerSel.value(state) === 0) {
      dispatch(timerEnd());
    } else { // not zero
      dispatch(); // ends process logic, nothing is dispatched
    }
  }
});
```

## Files of interest

 - [src/configureStore.js](./src/configureStore.js) - logicMiddleware is created with the combined array of logic for the app.

 - [src/rootLogic.js](./src/rootLogic.js) - combines logic from all other parts of the app and defines the order they appear in the logic pipeline. Shows how you can structure large apps to easily combine logic.

 - [src/timer/logic.js](./src/timer/logic.js) - the logic specific to the timer part of the app, this contains our timer logic

 - [src/timer/actions.js](./src/timer/actions.js) - contains the action creators

 - [src/timer/reducer.js](./src/timer/reducer.js) - contains a reducer which handles all the timer specific state. Also contains the timer related selectors. By collocating the reducer and the selectors we only have to update this one file to change the shape of our reducer state.

 - [src/timer/component.js](./src/timer/component.js) - Timer React.js component for displaying the status, timer value, and buttons (start, stop, reset)

 - [src/App.js](./src/App.js) - App component which uses redux connect to provide the polls state and bound action handlers as props

 - [test/timer-start-logic.spec.js](./test/timer-start-logic.spec.js) - testing timer start validation logic in isolation

## Usage

```bash
npm install # install dependencies
npm start # builds and runs dev server
```

Click start button which dispatches a simple `TIMER_START` action, that the logicMiddleware picks up, hands to `timerStartLogic` and runs the process hook starting a timer that is cancellable by receiving `TIMER_CANCEL`, `TIMER_RESET`, or `TIMER_END`. The timer will dispatch `TIMER_DECREMENT` actions every second while it is running.

Buttons for stop and reset issue the `TIMER_CANCEL` and `TIMER_RESET` actions.

The `timerDecrementLogic` validate hook will check whether the count is above zero and if so it will allow the action to proceed letting the reducer update the state. Also the process hook is also defined so we can check whether the state after the reducer updated is zero and then dispatch a `TIMER_END` action.
