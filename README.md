Greenlight
==========

[![CircleCI](https://circleci.com/gh/amperity/greenlight.svg?style=shield&circle-token=f934dddbf0f63d20e4df86f42f671b6f067227c2)](https://circleci.com/gh/amperity/greenlight)

This library provides an _integration testing_ framework for Clojure. Running
a suite of tests against your systems gives you the confidence to _greenlight_
them for promotion to production. The primary goals of this framework are:

- Steps should be _composable_ to drive down repetition in similar tests.
- TODO: Tests should be _distributable_ and support parallelization out-of-the-box.
- Results should be _actionable_ and easy to understand.


## Installation

Library releases are published on Clojars. To use the latest version with
Leiningen, add the following to your project dependencies:

[![Clojars Project](http://clojars.org/amperity/greenlight/latest-version.svg)](http://clojars.org/amperity/greenlight)


## Usage

A quick overview of greenlight usage:

```clojure
(require
  '[greenlight.test :as test :refer [deftest]]
  '[greenlight.step :as step :refer [defstep]]
  '[greenlight.runner :as runner]
  '[clojure.test :refer [is]])

;; Greenlight tests are built from a collection of steps

(defstep math-test
  "A simple test step"
  :title "Simple Math Test"
  :test (fn [_] (is (= 3 (+ 1 2)))))
=> #'user/math-test

(deftest simple-test
  "A simple test of addition"
  (math-test))
=> #'user/simple-test

;; Tests and steps are just data

(simple-test)
=> {:greenlight.test/description "A simple test of addition",
    :greenlight.test/line 14,
    :greenlight.test/ns user,
    :greenlight.test/steps [{:greenlight.step/inputs {},
                             :greenlight.step/name math-test,
                             :greenlight.step/test #<Fn@7bb15aaa user/math_test[fn]>,
                             :greenlight.step/title "Simple Math Test"}],
    :greenlight.test/title "simple-test"}


;; Tests can be ran individually

(test/run-test! {} (simple-test))
=> {:greenlight.test/context {},
    :greenlight.test/description "A simple test of addition",
    :greenlight.test/ended-at #<java.time.Instant@55e7469c 2018-07-01T17:03:29.811Z>,
    :greenlight.test/line 14,
    :greenlight.test/ns user,
    :greenlight.test/outcome :pass,
    :greenlight.test/started-at #<java.time.Instant@224450d6 2018-07-01T17:03:29.808Z>,
    :greenlight.test/steps [{:greenlight.step/cleanup [],
                             :greenlight.step/elapsed 0.002573744,
                             :greenlight.step/inputs {},
                             :greenlight.step/message "1 assertions (1 pass)",
                             :greenlight.step/name math-test,
                             :greenlight.step/outcome :pass,
                             :greenlight.step/reports [{:actual (3),
                                                        :expected 3,
                                                        :message nil,
                                                        :type :pass}],
                             :greenlight.step/test #<Fn@2be25eaa user/math_test[fn]>,
                             :greenlight.step/title "Simple Math Test"}],
    :greenlight.test/title "simple-test"}

;; Or as part of a suite with configurable reporters

(runner/run-tests! (constantly {}) [(simple-test)] {})

; Starting test system...
; Running 1 tests...
;
;  * Testing simple-test
;  | user:124
;  | A simple test of addition
;  |
;  +->> Simple Math Test
;  | 1 assertions (1 pass)
;  | [PASS] (0.000 seconds)
;  |
;  |
;  * PASS (0.001 seconds)
;
;
; Ran 1 tests containing 1 steps with 1 assertions:
; * 1 pass

=> true
```


### Test System

Tests are executed with a [system](https://github.com/stuartsierra/component)
of components. The component lifecycle is managed by the test runner: components
are started on test startup and then stopped on test completion.

```clojure
(require '[com.stuartsierra.component :as component])

(defrecord TestComponent
  []

  component/Lifecycle
  (start [this]
    (println "Starting test component")
    this)

  (stop [this]
    (println "Stopping test component")
    this))

(runner/run-tests!
  (constantly (component/system-map ::component (->TestComponent)))
  [(simple-test)]
  {})

; Starting test system...
; Starting test component
; Running 1 tests...
;
;  * Testing simple-test
;  | user:690
;  | A simple test of addition
;  |
;  +->> Simple Math Test
;  | 1 assertions (1 pass)
;  | [PASS] (0.000 seconds)
;  |
;  |
;  * PASS (0.001 seconds)
;
;
; Ran 1 tests containing 1 steps with 1 assertions:
; * 1 pass
;
; Stopping test component
=> true
```


### Step Inputs and Outputs

Test steps support parameterization through their inputs. Inputs can be statically
configured, pull components from the test system, or pull values built up from
previous steps in the test context.

Steps can additionally have outputs. These outputs are registered in the test
context and passed to subsequent steps. Outputs can be registered under a key,
a collection of keys, or as a function of the previous context.

```clojure
(defstep step-with-output-keyword
  "A step demonstrating a keyword output"
  :title "Keyword output"
  :test (constantly 1)
  :output :foo)

(defstep step-with-collection-output
  "A step demonstrating a nested output"
  :title "Nested output"
  :test (constantly 2)
  :output [:a :b :c])

(defstep step-with-inputs
  "A step demonstrating different input types"
  :title "Step with inputs"
  :inputs {;; For defaulting values
           :a 1
           ;; Extracting from test system
           :b (step/component ::component)
           ;; Extracting from test context
           :c (step/lookup :foo)
           ;; Context lookups can be a collection
           :d (step/lookup [:a :b :c])
           ;; Or for ultimate flexibility: a function
           :e (step/lookup (fn [ctx] (:bar ctx)))}
  :test (fn [{:keys [a b c d e]}]
          (is (every? some? [a b c d e]))))

(deftest inputs-and-outputs
  "A test demonstrating inputs and outputs"
  (step-with-output-keyword)
  (step-with-collection-output)
  ;; Steps can also be defined inline
  #::step{:title "function output"
          :name 'step-with-function-output
          :test (constantly 3)
          :output (fn [ctx test-output]
                    (assoc ctx :bar (* 2 test-output)))}
  (step-with-inputs
    ;; Override step inputs
    {:a :override-default}
    ;; Override step configuration
    :output :new-output
    :title "New Title"))

(runner/run-tests!
  (constantly (component/system-map ::component (->TestComponent)))
  [(inputs-and-outputs)]
  {})

; Starting test system...
; Starting test component
; Running 1 tests...
;
;  * Testing inputs-and-outputs
;  | user:36
;  | A test demonstrating inputs and outputs
;  |
;  +->> Keyword output
;  | 0 assertions ()
;  | [PASS] (0.000 seconds)
;  |
;  +->> Nested output
;  | 0 assertions ()
;  | [PASS] (0.000 seconds)
;  |
;  +->> Function output
;  | 0 assertions ()
;  | [PASS] (0.000 seconds)
;  |
;  +->> New Title
;  | 1 assertions (1 pass)
;  | [PASS] (0.000 seconds)
;  |
;  |
;  * PASS (0.003 seconds)
;
;
; Ran 1 tests containing 4 steps with 1 assertions:
; * 1 pass
;
; Stopping test component
=> true
```


### Step Cleanup

Often, steps will create some resource that should be removed on test completion.
The `step/clean!` multimethod along with `step/register-cleanup!` allows for test
steps to register resource specific cleanups. These cleanups are performed in
reverse order of registration.

```clojure
(defmethod step/clean! :foo/bar
  [system resource-type k]
  (printf "Removing %s resource %s" resource-type k)
  ;; Actual cleanup using system component
  )

(defstep step-with-cleanup
  "A step demonstrating resource cleanup"
  :title "Step with cleanup"
  :test (fn [_]
          (step/register-cleanup! :foo/bar :my-key)))

(deftest test-with-cleanup
  "A test demonstrating resource cleanup"
  (step-with-cleanup))

(runner/run-tests!
  (constantly {})
  [(test-with-cleanup)]
  {})

; Starting test system...
; Running 1 tests...
;
;  * Testing test-with-cleanup
;  | user:13
;  | A test demonstrating resource cleanup
;  |
;  +->> Step with cleanup
;  | 0 assertions ()
;  | [PASS] (0.000 seconds)
;  |
;  +->> Cleaning :foo/bar resource :my-key
; Removing :foo/bar resource :my-key |
;  * PASS (0.000 seconds)
;
; Ran 1 tests containing 1 steps with 0 assertions:
; * 1 pass
=> true
```


### Test Discovery

Tests can be specified explictly when running tests, or by utilizing
`runner/find-tests`, which retrieves tests based on a matcher. The matcher
can be either a keyword to match against test metadata or a regular expression
to match test name.

```clojure
(deftest ^:quick test-1
  ",,,"
  ,,,)

(deftest test-2
  ",,,"
  ,,,)

(runner/find-tests :quick)
=> (#:greenlight.test{:description ",,,", :title "test-1", :ns user, :line 1, :steps []})

(runner/find-tests #"test-2")
=> (#:greenlight.test{:description ",,,", :title "test-2", :ns user, :line 5, :steps []})
```

## License

Licensed under the Apache License, Version 2.0. See the [LICENSE](LICENSE) file
for rights and restrictions.
