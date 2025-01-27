[![Run abaplint](https://github.com/dominikpanzer/RESULT-for-ABAP/actions/workflows/lint.yml/badge.svg)](https://github.com/dominikpanzer/RESULT-for-ABAP/actions/workflows/lint.yml)
[![Run Unit Tests](https://github.com/dominikpanzer/RESULT-for-ABAP/actions/workflows/unittests.yml/badge.svg)](https://github.com/dominikpanzer/RESULT-for-ABAP/actions/workflows/unittests.yml)
[![Twitter](https://img.shields.io/twitter/follow/PanzerDominik?style=social)](https://twitter.com/PanzerDominik)

# RESULT for ABAP

Hi! "RESULT for ABAP" is - surprise, surprise - an ABAP implementation of the Result-Pattern. It's a way to solve a common problem: a method-call can be successful (OK) or it can fail (FAILURE) and the caller needs to know.  The Result-Pattern indicates if the operation was successful or failed without the usage of exceptions. It is a quite common pattern in functional languages.

## Give a Star! :star:
If you like or are using this project please give it a star. Thanks!

## Why should I use RESULT for ABAP instead of exceptions?
* Exceptions are actually only for... well, exceptional cases, like DB errors, locks etc. not for "domain errors" like validations etc. When you expect something (like the user entering wrong/invalid data) than it is actually no case for an exception. You actually seem to expect that users quite often are creative when entering data.
* Exception are often being used as a fancy form of the [GOTO-statement](https://en.wikipedia.org/wiki/Considered_harmful). You often don't know where they will be caught. If they get caught at all. This has been considered harmful in 1968.
* If a method can throw an exception and the calling code does not directly catch it - was this a mistake or on purpose? Will it be catch somewhere up the callstack? You never know, so have fun analyzing the whole stack.
* Exceptions lead to hard to read code, because regular business logic is mixed with technical error handling via TRY...CATCH-blocks, for example when many different exceptions have to be caught.
* Exceptions sometimes are not really helpful, because people tend to wrap all code into a TRY...CATCH-block for CX_ROOT. This can lead to inconsistent data, because a part of the data might already have been processed and persisted. It would be actually better to shortdump.
* Exceptions tend to return only one error, but what if you have multiple errors?
* Often command-methods just return a value like "it worked," which is either ABAP_TRUE or ABAP_FALSE. But no additional error values are available which could be shown in the frontend.
* Often query-methods just return the result of a query and if the result is empty, then this represents an error. But what was the reason for the error?
* Other methods export two values: the actual value and an optional error message. But now you can only use EXPORTING and not RETURNING, because there are two parameters. This leads to hard to read code. Ideally a method should only return one value.
* You could use a structure (value, error_message) to solve that problem. RESULT for ABAP is a comfortable object oriented way of doing this - a standardized solution. Your method simply returns a RESULT. 🦖
* RESULT enables a fluent coding style compared to TRY...CATCH...ENDTRY all over the place.
* Consistently using RESULT as the name for the returning parameter of methods simplifies method definitions and significantly improves readability of your code

## Okay, show me an example
### Creating successful RESULTs
```abap
* create a RESULT which represents a success (OK)
result = zcl_result=>ok( ).
* another one with additional information ("value", i.e. the key of an object you created or the object itself
result = zcl_result=>ok( 100040340 ).
* read the information again. You need to know the type of the value. that's the downside of a RESULT
* get_value can give you a reference, which works smoothly on new systems
DATA new_partner TYPE bu_partner.
new_partner = result->get_value( )->*.
* or it can give you direct access via the EXPORTING-Parameter, which is comfortable on 7.5x-systems
result->get_value( importing value = new_partner ).
* a result can be generated for example based on a validator-value. FAIL_IF will create a FAILURE,
* whenever the validator returns TRUE. when the validator returns FALSE, its an OK-RESULT
result = zcl_result=>fail_if( validator_returns_false( ) ).
* when a validator returns false and you can add an error message, it will be used if the RESULT is a FAILURE,
* otherwise its an OK-RESULT, like in this example
result = zcl_result=>fail_if( this_is_true = validator_returns_false( ) error_message = 'a wild error occurred' ).
* when a validator returns true, it will be an OK-RESULT
result = zcl_result=>ok_if( validator_returns_true( ) ).
* when a validator returns true its OK. but if the validator would fails, it is a FAILURE with an error message
result = zcl_result=>ok_if( this_is_true = validator_returns_true( ) error_message = 'a wild error occurred' ).
```
### Creating failures
```abap
* create a RESULT which indicates a FAILURE
result = zcl_result=>fail( ).
* with an error message
result = zcl_result=>fail('a wild error occurred').
* when a validator returns true
result = zcl_result=>fail_if( validator_returns_true( ) ).
* when a validator returns true + error message
result = zcl_result=>fail_if( this_is_true = validator_returns_true( ) error_message = 'a wild error occurred' ).
* when a validator returns false
result = zcl_result=>ok_if( validator_returns_false( ) ).
* when a validator returns false
result = zcl_result=>ok_if( this_is_true = validator_returns_false( ) error_message = 'a wild error occurred' ).
* reading an error message / the first error or all errors or all error messages
IF result->has_multiple_error_messages( ).
 DATA(error_message) = result->get_error_message( ).
ELSE.
 DATA(error_messages) = result->get_error_messages( ).
ENDIF.
* adding additional error messages to a FAILURE
zcl_result=>fail('a wild error occurred')->with_error_message( error_message ).
```
### Combining results
Usually there are many validations at the start of a method, so you might like to combine their single RESULTs into a final big one. The typical use case here is "validate X variables and all have to be OK, otherwise it's a FAILURE so stop processing the data". So if there is at least one FAILURE, the RESULT will be a FAILURE. Otherwise the RESULT will be OK. Currently only one error message will be stored. Combined OK-RESULTs don't have a value. You can also return a table of RESULTs from you method if you need the details.
```abap
* combined RESULT is OK
DATA(result_one) = zcl_result=>ok( ).
DATA(result_two) = zcl_result=>ok( ).
result = result_one->combine_with( result_two ).

* combined RESULT is a FAILURE
DATA results TYPE zcl_result=>ty_results.
DATA(result_one) = zcl_result=>ok( ).
DATA(result_two) = zcl_result=>fail( error_message ).
DATA(result_three) = zcl_result=>fail( error_message ).
results = VALUE #( ( result_two ) ( result_three ) ).
result = result_one->combine_with_these( results ).
```

### Adding Metadata to a RESULT
If you need more then just the one VALUE of an OK-RESULT, you can add metadata to the result. Metadata are key-value-pairs, with a unique CHAR30 key and the value being a data reference. Metadata can be added to any type of RESULT.
```abap
* Adding Metadata
DATA(structure) = VALUE zst_metadata_entry( key = 'a' value = REF #( 'random structure' ) ).
result = zcl_result=>ok( )->with_metadata( key = 'a structure' value = structure ).
result->with_metdata( key = 'band' value = 'Slayer' ).

* Reading metadata with a key
DATA(value) =  result->get_metadata( 'a structure' ).
* Reading the whole key-value-table
DATA(metadata) = result->get_all_metadata( ).
```

### Usage of a RESULT in a method
Use the RESULT as a RETURNING parameter:
```abap
METHOD do_something IMPORTING partner TYPE bu_partner
                    RETURNING VALUE(result) TYPE REF TO zcl_result.
                    
...

METHOD do_something.
* 100s of lines of arcane logic

* hooray, no problems at all
result = zcl_result=>ok( 100040340 ).
ENDMETHOD.
```
### Processing a RESULT
Use the RESULT-object for flow control as you like:
```abap
* call a method which returns a result
DATA new_partner TYPE bu_partner.
DATA(result) = do_something( partner ).

* guarding
IF result->is_failure( ).
DATA(error_message) = result->get_error_message( ).
* log / error for webservice
EXIT.
ENDIF.

new_partner = result->get_value( )->*.
* do something with partner, i.e. persistence
```

## How to install RESULT for ABAP
You can copy and paste the source code into your system or simply clone this repository with [abapGit](https://abapgit.org/). 

## Test List
I like to create a simple [acceptance test list](https://agiledojo.de/2018-12-16-tdd-testlist/) before I start coding. It's my personal todo-list. Often the list is very domain-centric, this one is quite technical.

|Test List|
|----|
:white_check_mark: first release somehow seems to works
:white_check_mark: when `FAIL_IF` gets called with an optional error message "a wild error occurred", the error message gets stored when the RESULT is a failure
:white_check_mark: when `FAIL_IF` has been called with an optional error message "a wild error occurred", `GET_ERROR_MESSAGE` will return "a wild error occurred" when the RESULT is a failure
:white_check_mark: when `FAIL_IF` has been called with an optional error message "a wild error occurred", `GET_ERROR_MESSAGE` will return an exception, when the RESULT is OK
:white_check_mark: when `OK_IF` gets called with an optional error message "a wild error occurred", the error message gets stored when the RESULT is a failure
:white_check_mark: when `OK_IF` has been called with an optional error message "a wild error occurred", `GET_ERROR_MESSAGE` will return "a wild error occurred" when the RESULT is a failure
:white_check_mark: when `OK_IF` has been called with an optional error message "a wild error occurred", `GET_ERROR_MESSAGE` will return an exception, when the RESULT is OK
:white_check_mark: when `OK_IF` has been called with an optional error message "a wild error occurred", `GET_VALUE` a initial value when the RESULT is OK
:white_check_mark: when a OK-RESULT uses a table as a VALUE, the VALUE can be retrieved and has the same number of lines as the original internal table
:white_check_mark: update the docs :japanese_ogre:
:white_check_mark: when the method `WITH_METADATA( key = "name" value = "David Hasselhoff" )` gets called once, the Metadata gets stored
:white_check_mark: when the method `GET_ALL_METADATA( )` gets called after `WITH_METADATA( key = "name" value = "David Hasselhoff" )`, it returns a table with one entry `(name, David Hasselhoff)`
:white_check_mark: when the method `GET_METADATA( name )` gets called after `WITH_METADATA( key = "name" value = "David Hasselhoff" )`, it returns a single value "David Hasselhoff"
:white_check_mark: when the method `GET_ALL_METADATA( )` gets called without `WITH_METDATA` being called before, it returns an initial table
:white_check_mark: when the method `GET_METADATA( date )` gets called after `WITH_METADATA( key = "name" value = "David Hasselhoff" )`, it returns an initial value
:white_check_mark: when the method `WITH_METADATA( key = "name" value = "David Hasselhoff" )` is called with the same key twice, no duplicates get stored and it throws
:white_check_mark: when the method `WITH_METADATA` is called with an initial key then that is okay
:white_check_mark: when the method `WITH_METADATA` is called twice with different keys `( key = "name" value = "David Hasselhoff" ) ( key = "name2" value = "David Hasselhoff" )`, both values get stored
:white_check_mark: when the method `WITH_METADATA( key = "name" value = value )` and value a structure, a structure will be returned by get_metadata( name ).
:white_check_mark: update the docs :japanese_ogre:
:white_check_mark: when `COMBINE_WITH` gets called with two failures, both error messages get stored
:white_check_mark: when `COMBINE_WITH_THESE` gets called with tow failures, both error messages get stored
:white_check_mark: when `GET_ERROR_MESSAGES` gets called for an FAILURE with two error messages, it returns two error messages
:white_check_mark: when `GET_ERROR_MESSAGE` gets called on a FAILURE it returns only the first error message
:white_check_mark: `HAS_MULTIPLE_ERROR_MESSAGES` returns false when there is no error_message for a FAILURE
:white_check_mark: `HAS_MULTIPLE_ERROR_MESSAGES` returns false when there is only one error_message for a FAILURE
:white_check_mark: `HAS_MULTIPLE_ERROR_MESSAGES` returns true when there a multiple error_messages for a FAILURE
:white_check_mark: `HAS_MULTIPLE_ERROR_MESSAGES` throws when OK-RESULT
:white_check_mark: `GET_ERROR_MESSAGE` throws when OK-RESULT
:white_check_mark: when `WITH_ERROR_MESSAGE gets called with an empty message on a FAILURE, it just returns the current result
:white_check_mark: when `WITH_ERROR_MESSAGE( 'a wild error occurred' )` gets called on a FAILURE, the message will be added to the list of error messages and can be retrieved with `GET_ERROR_MESSAGES`
:white_check_mark: when `WITH_ERROR_MESSAGE( 'a wild error occurred' )` gets called on a OK-RESULT it does not do anything but return the result
:white_check_mark: update the docs :japanese_ogre:
:white_check_mark: get_value has `RETURNING` and `EXPORTING` parameters to make handling easier on 7.5x-systems
:black_square_button: your awesome idea

As you can see I'm usually committing after every green test or even more often. I tend to use the ´zero, one, multiple´ or the ´happy path, unhappy path´ patterns to write my tests to drive my logic.

## How to support this project

PRs are welcome! You can also just pick one of the test list entries from above and implement the solution or implement your own idea. Fix a bug. Improve the docs... whatever suits you.

Greetings, 
Dominik

follow me on [Twitter](https://twitter.com/PanzerDominik) or [Mastodon](https://sw-development-is.social/web/@PanzerDominik)


