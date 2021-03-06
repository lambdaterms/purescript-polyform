# purescript-polyform

An attempt to build simple, composable validation toolkit.

## Objectives

  * Be completely render backend agnostic (form can be rendered as HTML but also as for example... HTTP query).

  * Allow validation to be abstracted over data source, so we can reuse it on the backend or frontend or during typed input validation.

  * But also provide modules dedicated for handling `HTTP` input or `Foreign` data.

  * Validation should produce well typed result value __and form__ (aka "report") structure or form filled with errors, so you can always render it.

  * Exact structure of form and fields are up to the user. The main constraint is that form should be a `Monoid` (it may sound limiting at first but there are also monoids like... `Join (->) a` ;-)).

  * Provide only minimal representation of form fields with attributes which are relevant to validation.

## Validation overview

### Types

  In `Polyform.Validation` you can find `Validation` type - a function which in case of a success produces final result and a monoidal "validation report" value. In case of a validation error it also produces value of this "report" type as a failure representation. Our validation result has type:

  ```purescript
    data V r a = Invalid r | Valid r a
  ```
You can think about this `r` as a carrier for some informations you want to produce from validation beside your final value. This "report" possibly contains details about failed or successful processing, but also can carry informations about your validation process (for example marking "pending" validation etc.) or some extra data (like Html field attributes). Some examples:

* In case of traditional HTML forms it is often a requirement to render a form even if it is completely valid. Or render all valid and invalid fields.
In such a case your "report" would be possibly list of fields (some valid and some invalid) and list of form level errors (or even nested subforms with errors). This kind of representation can be easily composable by monoidal `append`.

* When you are validating `Foreign` value you can build representation of your possibly nested tree of objects in your monoidal report and indicate exactly which parts failed to validate (please check `Polyform.Input.Foreign` if you are interested).

* Your `r` can also be this strange `a -> a` `Monoid` and represent some transformation (for example a record representation of your form). Simple example can be found [here](https://github.com/paluh/purescript-polyform-record-form-example).


Going back to `Validation` function. It is just a function with additional `Monadic` context `m` which produces above `V`:

  ```purescript
    data Validation m e q a = Validation (q -> m (V e a))
  ```
  We can think of `q` as an input data/query, `m` as a computational context, `e` could be our "report" and `a` is a result type of successful validation.

  Having this structure of validation we can combine (using `Applicative` or `Alt` or `Category` instances - all of them just append our monoidal values) multiple validation functions to produce larger and larger forms even when some of these functions fail.

  All combined (using `Applicative` or `Alt`) validation functions operate on the same input data in similar way as `Applicative` instance is implemented for `Function` type.

  Values glued together through `Category` composition form a processing chain. Inside this chain you can always append "errors" (or any information) to your "form" value. In this manner we are able to perform additional checking, validation etc.

### Quick example

To gain some intuition about this library desing and how this architecture of validation works in practice we are going to build record validation backend and some forms from scratch - without any ready to use "helpers" or backends from polyform, but using only `Validation`.

As we want to validate records we have to somehow fetch values from them pass them to validation functions and accumulate errors or return a result. But how to access a record field... maybe with field accessor - a function like this: `_.myField` (thanks @thomashoneyman)!
This solves our most difficult problem for this backend so let's write some code:

```purescript
module Main where

import Prelude

import Control.Monad.Eff (Eff)
import Control.Monad.Eff.Console (CONSOLE, log)
import Control.Monad.Eff.Random (random)
import Data.Array (any, elem)
import Data.Maybe (Maybe(..))
import Data.String (Pattern(..), contains, indexOf, length, toCharArray)
import Data.Tuple (Tuple(..))
import Data.Variant (Variant, inj)
import Debug.Trace (traceAnyA)
import Polyform.Validation (V(..), Validation(..), runValidation)
import Polyform.Validation as Validation
import Type.Prelude (SProxy(..))


-- | Let's assume that our fields are really simple
-- | and contain only validation result or
-- | errors which are kept in `Array`.
type Input err value = V (Array err) value

-- | Let's define some simple validators for email field

-- | ...of course they are really dummy validators ;-)

-- | `Validation.hoist*` transform functions to `Validation`

emailFormat = Validation.hoistFnV \e →
  -- | @ is just enough for as to send an email ;-)
  if contains (Pattern "@") e
    then pure e
    else Invalid [inj (SProxy ∷ SProxy "emailFormat") e]

emailIsUsed = Validation.hoistFnMV \e → do
  -- | Some effectful computation inside your monad.
  -- | Let's toss a coin instead of quering the db
  -- | if email is really used :-P
  v ← random
  pure $ if v > 0.5
    then Invalid [inj (SProxy ∷ SProxy "emailIsUsed") e]
    else pure e

-- | Let's define some simple validators for password field.

minLength m = Validation.hoistFnV \p →
  if length p < m
    then Invalid [inj (SProxy ∷ SProxy "minLength") (Tuple m p)]
    else pure p

maxLength m = Validation.hoistFnV \p →
  if length p > m
    then Invalid [inj (SProxy ∷ SProxy "maxLength") (Tuple m p)]
    else pure p

hasDigit = Validation.hoistFnV \p →
  let
    chars = toCharArray p
  in
    if any (_ `elem` chars) (toCharArray "0123456789")
      then pure p
      else Invalid [inj (SProxy ∷ SProxy "hasDigit") p]

-- | Here we are combining validations using categorical
-- | composition so first validation has to succeed
-- | to second could proceed on it's result.
emailFieldValidation = emailFormat >>> emailIsUsed

-- | We are combining validations using applicative
-- | instance which in essence works like that:
-- |
-- | pure (Valid e1 a1) *> pure (Invalid e2) = Invalid (e1 <> e2)
-- | pure (Invalid e1) *> pure (Valid e2 a2) = Invalid (e1 <> e2)
-- | pure (Valid e1 a1) *> pure (Valid e2 a2) = Valid (e1 <> e2) a2
-- |
passwordFieldValidation min max = maxLength max *> minLength min *> hasDigit

-- | It is worth to point out that there is also `Alt` instance
-- | which in essence works like that:
-- |
-- | pure (Valid e1 a1) <|> pure (Invalid e2) = Valid (e1 <> e2) a1
-- | pure (Valid e1 a1) <|> pure (Valid e2 a2) = Valid (e1 <> e2) a1
-- |
-- | It is not used here so sorry for this spam...

-- | Now we are ready to define field type

data Field
  = EmailField
      (Input (Variant (emailFormat ∷ String, emailIsUsed ∷ String)) String)
  | PasswordField
      (Input (Variant (hasDigit ∷ String, maxLength ∷ Tuple Int String, minLength ∷ Tuple Int String)) String)


-- | This is our form type so when you see `Tuple`
-- | below it means that we are building a Form.
-- |
-- | Such a `Tuple` gives us `Monoid` for free.
type Form = Tuple (Array String) (Array Field)

-- | Let's build our form without any external helpers
-- | This function builds a single field form based on:
-- |  * value fetcher
-- |  * field type constructor
-- |  * field validation
-- |
-- | Here we can also observe that validation is
-- | nothing more than a function from an input to
-- | some `V` value in monadic context.
fieldForm fetchValue constructor fieldValidation =
  Validation $ \inputRecord → do
    -- | Fetch field value from record using fetcher
    let inputValue = fetchValue inputRecord
    -- | Run field validation agains this value
    r ← Validation.runValidation fieldValidation inputValue
    -- | Based on field validation result let's return:
    pure $ case r of
      -- | form togheter with result value
      -- | so we can combine both into larger values and forms
      Valid e v → Valid (Tuple [] [constructor (Valid e v)]) v
      -- | or form as representation of our error which
      -- | can be combined with other forms
      Invalid e → Invalid (Tuple [] [constructor (Invalid e)])

emailForm = fieldForm (_.email) EmailField emailFieldValidation

buildPasswordForm fetch = fieldForm fetch PasswordField (passwordFieldValidation 5 50)

-- | This form introduces different level of validation.
-- | Field validations errors are accumulated in our specific case
-- | in on the field level (in the `Array` of `errs`) as defined above.
-- |
-- | Here we introducing form level errors which are aggregated
-- | on the form level inside this first `Array` of `Strings` from
-- | our form `Tuple`.
-- |
-- | I'm talking about the second part of this function...
passwordForm
  = ({password1: _, password2: _} <$> (buildPasswordForm _.password1) <*> (buildPasswordForm _.password2))
  -- | Here we are composing validations
  -- | so previous step results
  -- | (record with two passwords)
  -- | are inputs for this next step.
  -- |
  -- | This is nearly a "function composition"
  -- | (or The Kleis...;-)
  -- | so you can always return from this function
  -- | completely new type. For example here
  -- | we are returning single password value.
  -- |
  -- | Of course this is composition of validation
  -- | function so you can use your monadic context
  -- | here too.
  -- |
  -- | We can always fail here and return
  -- | form representing our failure
  -- | which will be appended to the
  -- | whole form.
  >>> Validation.hoistFnV \{ password1, password2 } →
    if password1 /= password2
      then Invalid (Tuple ["Password dont match"] [])
      else pure password1

signupForm = {password: _, email: _} <$> passwordForm <*> emailForm

printResult =
  case _ of
    Valid form value → do
      log "FORM VALID:"
      traceAnyA form
      log "FINAL VALUE:"
      traceAnyA value

    Invalid form → do
      log "FORM INVALID:"
      traceAnyA form

main = do
  log "EXAMPLE"

  v1 ← runValidation signupForm {email: "wrongemailformat", password1: "shrt", password2: "nodigits"}
  printResult v1

  log "\n\n"

  v2 ← runValidation signupForm {email: "email@example.com", password1: "password1", password2: "password2"}
  printResult v2

  log "\n\n"

  v3 ← runValidation signupForm {email: "email@example.com", password1: "password921", password2: "password921"}
  printResult v3
```

And here is raw output from the above program (of course you result can be different as
we using coin tossing as an email validation method :-P):

```purescript
Tuple {
  value0: [],
  value1:
   [ PasswordField {
       value0:
        Invalid {
          value0:
           [ { type: 'minLength',
               value: Tuple { value0: 5, value1: 'shrt' } },
             { type: 'hasDigit', value: 'shrt' } ] } },
     PasswordField {
       value0: Invalid { value0: [ { type: 'hasDigit', value: 'nodigits' } ] } },
     EmailField {
       value0:
        Invalid {
          value0:
           [ { type: 'emailFormat', value: 'wrongemailformat' },
             { type: 'emailIsUsed', value: 'wrongemailformat' } ] } } ] }


FORM INVALID:
Tuple {
  value0: [ 'Password dont match' ],
  value1:
   [ PasswordField { value0: Valid { value0: [], value1: 'password1' } },
     PasswordField { value0: Valid { value0: [], value1: 'password2' } },
     EmailField { value0: Valid { value0: [], value1: 'email@example.com' } } ] }



FORM VALID:
Tuple {
  value0: [],
  value1:
   [ PasswordField { value0: Valid { value0: [], value1: 'password921' } },
     PasswordField { value0: Valid { value0: [], value1: 'password921' } },
     EmailField { value0: Valid { value0: [], value1: 'email@example.com' } } ] }
FINAL VALUE:
{ password: 'password921', email: 'email@example.com' }
```

### Parallel validation

There is simple wrapper which allows you to execute validations in "parallel" using your underling monad parallelism - check `Polyform.Validation.Par`. Shortly you can build parrallel execution of validation tree using (`alt` or `apply`) for example like this:

```purescript
sequential $ { email: _, password: _} <$> parallel emailForm <*> parallel passwordForm
```

You have to use `sequential` and `parallel` from `Polyform.Validation.Par` as I'm not able to implement `Parallel` instance for non monad (`Validation` doesn't form a `Monad`).


## Project structure

* `Validation` is the core part of this library where validation type is defined and it's parallel wrapper.

* `Field` contains some proposition which can be a base for `Html` (especially `Field.Html5`) form validation. Other types of validation could also reuse parts of this helpers. You can find there also some generic solutions for choice and multichoice validation.

* `Form` provides two types of components.

    * First builds up default form value when composed.

    * Second also composes serialization function. You can generate input for your form from your final value. Be careful it generated input doesn't have to be "valid" (you should probably mark input elements which you don't want to validate) ;-)

* `Input` is a battle field which explores validation scenarios for different types of input. Currently we have: `Input.Foreign`, `Input.Http` and `Input.Interpret`.

## API Documentation

Module documentation is [published on Pursuit](http://pursuit.purescript.org/packages/purescript-polyform).
