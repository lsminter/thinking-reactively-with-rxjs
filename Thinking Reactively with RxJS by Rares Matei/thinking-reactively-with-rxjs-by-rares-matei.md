## [Github Prep:](https://github.com/rarmatei/thinking-reactively-workshop.git)

This is a link to the repo that we will be following along in the live workshop. 

## Post Workshop Notes:

Great workshop overall! I really enjoyed following along with this one. It was easy to follow along and understand everything that the instructor taught. I only had one or two questions that I asked down below in my LIVE WORKSHOP NOTES. 

He did a great job answering questions. I don't think that a single question went unanswered. 

I missed a little bit because I needed to run to the restroom at the office which takes a little bit to do but because of how he set up the repo, I could hop right back in where he was and not have to worry about trying to catch up. 

## Live Workshop Notes:

RXJS sits above vanilla JS. Makes it very easy for humans to read. 

We will be working in a small mobile application. We are going to building a loading spinner. 

Starting in `TaskProgressService.js` 

```js
main requirement - when there are async tasks in the background, show a loading spinner on screen

when the loader to show
    -> show the loader until it's time to hide it

when does the loader need to show? 
    when the count of async tasks goes from 0 to 1

when does the loader need to hide? 
    when the count goes to zero

how do we count the async tasks? 
    start from zero
    when an async task starts, increase the count by 1
    when a task ends, decrease the count by one

unknowns: 
    when an async task starts
    when an async task ends
    how to show the loader on the screen
```

You then want to create three observables that we aren't too sure of how we are going to use them. 

```js
const taskStarts = new Observable();
const taskCompletions = new Observable();
const displaySpinner = new Observable();
```

Lost audio here for a bit. Had to do some trouble shooting to get audio back. Continued to type along. 
```js
const taskStarts = new Observable();
const taskCompletions = new Observable();
const displaySpinner = new Observable();

const LoadUp = taskStarts.pipe(mapTo(1));
const LoadDown = taskCompletions.pipe(mapTo(-1));
const loadVariations = merge(loadUp, LoadDown);
```

I like that the instructor had it already broken down in different sections so that if someone got lost or missed something, they can look at the code at the part they are at and immediately catch back up. 

We now start working on creating our counter. 

```js
const currentLoadCount = LoadVariations.pipe(
    scan((totalCurrentsLoads, changeInLoads) => {
        const newLoadcount = totalCurrentsLoads + changeInLoads
        return newLoadcount;
    })
)
```

If someone is using our app incorrectly and making the tasks go below 0, we can fix that here in the counter: 

```js
return newLoadcount > 0 ? newLoadcount : 0;
```

When you subscribe, we want to give them an initial value of 0. 

```js
const currentLoadCount = LoadVariations.pipe(
    scan((totalCurrentsLoads, changeInLoads) => {
        const newLoadcount = totalCurrentsLoads + changeInLoads
        return newLoadcount > 0 ? newLoadcount : 0;
    }),
    startWith(0)
)
```

It does matter where you put your `startWith`. 

if you put it before the scan, then the scan will also get that value, if you do it after the scan, only the subscriber will get it. 

You can give it a default though: 

```js
const currentLoadCount = LoadVariations.pipe(
    scan((totalCurrentsLoads, changeInLoads) => {
        const newLoadcount = totalCurrentsLoads + changeInLoads
        return newLoadcount > 0 ? newLoadcount : 0;
    }, 0),
    startWith(0)
)
```

—BREAK—

Whenever we get a value from the observable, it will make sure that it is different than the value before it. To do this we use `distinctUntilChanged`. 

```js
const currentLoadCount = LoadVariations.pipe(
    scan((totalCurrentsLoads, changeInLoads) => {
        const newLoadcount = totalCurrentsLoads + changeInLoads
        return newLoadcount > 0 ? newLoadcount : 0;
    }, 0),
    startWith(0),
    distinctUntilChanged()
)
```

...........

Time to revise: 

```js
const currentLoadCount = loadVariations.pipe(
    scan((totalCurrentLoads, changeInLoads) => {
    const newLoadCount = totalCurrentLoads + changeInLoads;
    return newLoadCount > 0 ? newLoadCount : 0;
    }, 0),
    startWith(0),
    distinctUntilChanged(),
    shareReplay(1)
);

const shouldHideSpinner = currentLoadCount.pipe(
    filter(count => count === 0)
);

const shouldShowSpinner = currentLoadCount.pipe(
    pairwise(),
    filter(([prevCount, currCount]) => currCount === 1 && prevCount === 0)
);

shouldShowSpinner
    .pipe(switchMap(() => displaySpinner.pipe(takeUntil(shouldHideSpinner))))
    .subscribe();

export default {};
```

`Subject` is really useful because they are an observer and observable. They are observables that you can put stuff on. 

```js
const taskStarts = new Subject();
const taskCompletions = new Subject();

const loadUp = taskStarts.pipe(mapTo(1));
const loadDown = taskCompletions.pipe(mapTo(-1));

const loadVariations = merge(loadUp, loadDown);

const currentLoadCount = loadVariations.pipe(
    scan((totalCurrentLoads, changeInLoads) => {
    const newLoadCount = totalCurrentLoads + changeInLoads;
    return newLoadCount > 0 ? newLoadCount : 0;
    }, 0),
    startWith(0),
    distinctUntilChanged(),
    shareReplay(1)
);

const shouldHideSpinner = currentLoadCount.pipe(filter(count => count === 0));

const shouldShowSpinner = currentLoadCount.pipe(
    pairwise(),
    filter(([prev, curr]) => curr === 1 && prev === 0)
);

const shouldShowWithDelay = shouldShowSpinner.pipe(
    switchMap(() => {
    return timer(2000).pipe(takeUntil(shouldHideSpinner));
    })
);

shouldShowWithDelay
    .pipe(switchMap(() => displaySpinner().pipe(takeUntil(shouldHideSpinner))))
    .subscribe();

function displaySpinner(total, loaded) {
    return new Observable(() => {
    const loadingSpinnerInstance = initLoadingSpinner(total, loaded);
    loadingSpinnerInstance.then(spinner => spinner.show());
    return () => {
        loadingSpinnerInstance.then(spinner => spinner.hide());
    };
    });
}

export function newTaskStarted() {
    taskStarts.next();
}

export function existingTaskCompleted() {
    taskCompletions.next();
}

export default {};
```

We also now have a spinner that starts when there is a task and disappears when the task is gone. 

—BREAK—

Going to work on fixing some lag. 

We are going to delay the timer for 2 seconds before showing it. When the task is done, it will immediately disappear. 

```js
const shouldShowWithDelay = shouldShowSpinner.pipe(
    switchMap(() => {
    return timer(2000).pipe(takeUntil(shouldHideSpinner));
    })
);

shouldShowWithDelay
    .pipe(switchMap(() => displaySpinner().pipe(takeUntil(shouldHideSpinner))))
    .subscribe();
```

When do we need to hide the spinner? When two events have happened: when we need to hide the spinner immediately and when 2 seconds have passed.

To fix this we make it so that if a task takes longer than 2 seconds to complete, it will automatically show a spinner for at least two seconds. 

```js
const flashThresholdMs = 2000;

const shouldShowWithDelay = shouldShowSpinner.pipe(
    switchMap(() => {
    return timer(flashThresholdMs).pipe(takeUntil(shouldHideSpinner));
    })
);

const shouldHideWithDelay = combineLatest(
    shouldHideSpinner.pipe(first()),
    timer(flashThresholdMs)
);

shouldShowWithDelay
    .pipe(switchMap(() => displaySpinner().pipe(takeUntil(shouldHideWithDelay))))
    .subscribe();

function displaySpinner(total, loaded) {
    return new Observable(() => {
    const loadingSpinnerInstance = initLoadingSpinner(total, loaded);
    loadingSpinnerInstance.then(spinner => spinner.show());
    return () => {
        loadingSpinnerInstance.then(spinner => spinner.hide());
    };
    });
}
```

We created the `flashThresholldMs` variable so that we can just use that instead of repeating the `2000` for each of the timers. 

Really liked the `a`, `s`, `d`, `f` example that he showed. Was really cool. Started working on this in a new file called `KeyCombo.js`. 

```js
/* 
Whenever the user starts a combo
    Keep taking(Listening) for the rest of the combo keys
        until the timer has finished
        while the combo is being followed correctly
        and until we got size-of-combo - 1 keys pressed.
*/

const anyKeyPresses = fromEvent(document, "keypress").pipe(
    map(event => event.key)
);

function keyPressed(key) {
    return anyKeyPresses.pipe(filter(keyPressed => keyPressed === key))
}

export function keyCombo(keyCombo) {
    const comboInitiator = keyCombo[0];
    return keyPressed(comboInitiator)
}

const okPresses = keyCombo(['o', 'k']);
```

In COMBO MODE, we want to start tracking the key presses. 

```js
export function keyCombo(keyCombo) {
    const comboInitiator = keyCombo[0];
    return keyPressed(comboInitiator)
        .pipe(switchMap(() => {
            //COMBO MODE
            return anyKeyPresses
        }))
}
```

Keep taking combo keys until the timer has finished. 

```js
export function keyCombo(keyCombo) {
    const comboInitiator = keyCombo[0];
    return keyPressed(comboInitiator)
        .pipe(switchMap(() => {
            //COMBO MODE
            return anyKeyPresses.pipe(
                takeUntil(timer(5000))
            )
        }))
}
```

The second condition, keep taking while the combo is being followed correctly. For that, we use `takeWhile`. 

```js
export function keyCombo(keyCombo) {
    const comboInitiator = keyCombo[0];
    return keyPressed(comboInitiator)
        .pipe(switchMap(() => {
            //COMBO MODE
            return anyKeyPresses.pipe(
                takeUntil(timer(5000))
                takeWhile((keyPressed, index) => )
            )
        }))
}
```

We need the index to check if the key in our combo matches what we want it to match with. 

```js    
    export function keyCombo(keyCombo) {
        const comboInitiator = keyCombo[0];
        return keyPressed(comboInitiator)
            .pipe(switchMap(() => {
                //COMBO MODE
                return anyKeyPresses.pipe(
                    takeUntil(timer(5000)),
                    takeWhile((keyPressed, index) => keyPressed === keyCombo[index + 1])
                )
            }))
    }
```

For the last thing, we listen until we got size-of-combo minus 1 keys pressed.

```js
export function keyCombo(keyCombo) {
    const comboInitiator = keyCombo[0];
    return keyPressed(comboInitiator)
        .pipe(switchMap(() => {
            //COMBO MODE
            const innerComboSize = keyCombo.Length - 1;
            return anyKeyPresses.pipe(
                takeUntil(timer(5000)),
                takeWhile((keyPressed, index) => keyPressed === keyCombo[index + 1]),
                take(innerComboSize)
            )
        }))
}
```

This will give us a notification for every key we press. We just want one notification when we finish the combo. 

Finished product: 

```js
const anyKeyPresses = fromEvent(document, "keypress").pipe(
    map(event => event.key)
    );
    
    function keyPressed(key) {
    return anyKeyPresses.pipe(filter(pressedKey => pressedKey === key));
    }
    
    export function keyCombo(keyCombo) {
    const comboInitiator = keyCombo[0];
    return keyPressed(comboInitiator).pipe(
        exhaustMap(() => {
        return anyKeyPresses.pipe(
            takeWhile((key, index) => keyCombo[index + 1] === key),
            skip(keyCombo.length - 2),
            take(1),
            takeUntil(timer(5000))
        );
        })
    );
    }
```

Now we are going to create some helpers. 

What if we can have an operator that tracks anything above it. 

We are going to pass the `showLoadingStatus` operator onto the `pipe` function. 

    const slowObservable = timer(3000).pipe(showLoadingStatus());

We are going to create `showLoadingStatus` by creating a new file called `Extensions.js`. 

```js
import { existingTaskCompleted, newTaskStarted} from './TaskProgressService'
import { Observable } from 'rxjs';

export function showLoadingStatus() {
    return source => {
        return new Observable(() => {
            newTaskStarted();
            
        });
    }
}
```

The source needs to be unchanged. To fix that, we subscribe to the source. Inside of the subscribe block, we add our functions, next, error, and complete. 
```js
export function showLoadingStatus() {
    return source => {
        return new Observable(() => {
            newTaskStarted();
            source.subscribe({
                next: () => {},
                error: () => {},
                complete: () => {},
            })
        });
    }
}
```
Working through those three functions:

```js
export function showLoadingStatus() {
    return source => {
        return new Observable(subscriber => {
            newTaskStarted();
            source.subscribe({
                next: () => subscriber.next(val),
                error: error => {
                    subscriber.error(err);
                },
                complete: () => {
                    existingTaskCompleted();
                    subscriber.complete();
                },
            })
        });
    }
}
```

Now we need to be able to unsubscribe.