  # Processing of unknown enum values by SDL Core
  * Proposal: [SDL-0298](0298-Processing-of-unknown-enum-values-by-SDLCore.md)
  * Authors: [Igor Gapchuk](https://github.com/IGapchuk), [Dmitriy Boltovskiy](https://github.com/dboltovskyi/)
  * Status: **Awaiting Review**
  * Impacted Platforms: [SDL Core]

## Introduction
The main purpose of this proposal is to gracefully handle unknown enum values from a mobile RPC request with any given SDL Core version. This can be achieved by improving the way SDL Core processes and validates unknown enum values.

## Motivation

The current filtering and data validation mechanism used by SDL Core needs be improved upon to issues related to possible differences in API versions between parameters in the request from a mobile application and an SDL Core.

To better understand what kinds of issues we could face, let's assume the next example:

A mobile device with a newer version of the API (5.0) during connecting to the SDL Core, which has an older API version (4.3), sends a `RegisterAppInterface` request to SDL Core with an array of `AppHMITypes` which has the next values: `[DEFAULT, REMOTE_CONTROL]` (The `AppHMIType::DEFAULT` was introduced in the API since 2.0 version, since the time the `AppHMIType` type was added into the API, and `AppHMIType::REMOTE_CONTROL` was introduced in the 4.5 version of the API). SDL Core tries to validate the parameters in the RAI request, finds an unknown enum value of `AppHMIType::REMOTE_CONTROL`. As a result, the SDL Core sends a failure response to a mobile application with the `INVALID_DATA` result code. Thus, the mobile application fails to register with SDL Core that is an unacceptable user experience. Besides, the mobile application cannot conclusively determine the cause of failure since `INVALID_DATA` can be returned for a line of reasons.
With the changes provided by this proposal, SDL Core will be able to process all unknown enums in mobile requests in the correct way and respond to mobile applications with informative messages about happed failures.

## Proposed solution

When SDL Core receives a request from a mobile application, it starts the validation of all incoming parameters.
If a parameter includes an unknown enum value, SDL Core has to cut off such value from the parameter.

  This means the following cases are possible:

  1. The type of parameter is a non-array enum (see example 1).

      SDL Core has to remove this parameter from the request since the value after cutting off becomes empty.

  2. A parameter is an array of enum types (see example 2).

      SDL Core has to remove an unknown value from the array.
      A case is possible when all values are unknown and thus get removed.
      In this case, SDL Core has to proceed the same way as in case 1.

  3. A parameter is part of the structure (see example 3).

      SDL Core has to proceed the same way as in case #1 or #2 and remove this parameter from the structure.
      However, if the parameter is mandatory the structure becomes invalid.
      In this case, SDL Core has to remove the whole structure from the request.
      During this process, SDL Core has to proceed recursively from the very bottom level up to the top.

  4. A parameter is a part of the structure which is a part of an array (see example 4).

      SDL Core has to process it the same way as in case #3 and remove the structure as an item of the array.
      Once all the parameters are processed SDL Core has to proceed with the request as usual.

  If at least one value of at least one parameter was cut off, SDL Core has to update the response to the mobile application as follows:

  1. If the response was processed successfully, SDL Core has to provide the `WARNINGS` result code instead of `SUCCESS`.
  2. SDL Core has to provide removed enum items in the `info` string of the response.

      Since there could be more than one parameter with cut-off value, `info` message can be constructed the following way:

      `Invalid enums were removed: <param_1>:<enum_value_1>,<enum_value_2>;<param_2>:<enum_value_3> ...`

      If the `info` parameter contains other value(s) belonging to the original processing result SDL Core has to append information about cut-off enum value(s) to the existing value.

  **Examples:**
  In each example below the mobile application sends a request to SDL Core.
  1. The request contains a parameter which value is an enum item.

      Request: `SetMediaClockTimer`

      1.1 Parameter is mandatory: `updateMode="UNKNOWN"`.

      SDL Core removes the item `UNKNOWN` from the request.
      The value of the `updateMode` parameter becomes empty.
      SDL Core omits an empty parameter from the request.
      Since the parameter is mandatory (according to the SDL Core API), the request fails with the following response to a mobile application:
      `success=false, resultCode = "INVALID_DATA", info="Invalid enums were removed: updateMode:UNKNOWN"`

      1.2 Parameter is optional: `audioStreamingIndicator="UNKNOWN"`.

      SDL Core removes the `UNKNOWN` item from the request.
      The value of the `audioStreamingIndicator` parameter becomes empty.
      SDL Core omits the empty parameter in the request.
      Since the parameter is optional (according to the SDL Core API), the request is successfully processed with the following response to a mobile application:
      `success=true, resultCode = "WARNINGS", info="Invalid enums were removed: audioStreamingIndicator:UNKNOWN"`

  2. The request contains a parameter for which the value is an array of enum items.

      Request: `RegisterAppInterface`.

      2.1 Only one enum item is unknown: `appHMIType = [ "DEFAULT", "UNKNOWN"]`.

      SDL Core removes the `UNKNOWN` item from the request.
      The value of the `appHMIType` parameter now contains only `DEFAULT` item (which is known one).
      The request is successfully processed with the following response to a mobile application:
      `success=true, resultCode = "WARNINGS", info="Invalid enums were removed: appHMIType:UNKNOWN"`

      2.2 All enum items are unknown: `appHMIType = [ "UNKNOWN_1", "UNKNOWN_2"]`.

      SDL Core removes both `UNKNOWN_1` and `UNKNOWN_2` items from the request.
      The value of `appHMIType` parameter becomes empty.
      Since `minsize` of the parameter is `1` (according to the SDL Core API), the request fails with the following response to mobile application:
      `success=false, resultCode = "INVALID_DATA", info="Invalid enums were removed: appHMIType:UNKNOWN_1, UNKNOWN_2"`

  3. The request contains a parameter (with enum item value) which is a part of the structure.

      Request: `SetGlobalProperties`.

      Parameter `menuIcon="Image"`, sub-parameter: `imageType`, `type="ImageType"`

      ```
      menuIcon = {
        imageType = "UNKNOWN"
      }
      ```

      SDL Core removes the `menuIcon` parameter from the request since the parameter `imageType` is mandatory for the `ImageType` structure.
      However, request will be processed successfully since the `menuIcon` parameter is optional for `SetGlobalProperties` request and the following response will be provided to mobile application:

      `success=true, resultCode = "WARNINGS", info="Invalid enums were removed: menuIcon.imageType:UNKNOWN"`

  4. The request contains a parameter which is an array of elements and each element has a parameter which value is an enum item.

      Request: `Show`.

      Parameter: `softButtons`, `type="SoftButton"`, sub-parameter: `type`, `type="SoftButtonType"`.

      There are 2 soft buttons provided in the request:

      ```
      softButtons = [
        { softButtonID = 1, type = "UNKNOWN" },
        { softButtonID = 2, type = "IMAGE" }
      ]
      ```

      SDL Core removes the whole element with the button `1` from the request since the `type` parameter is mandatory for `SoftButton` structure:

      ```
      softButtons = [
        { softButtonID = 2, type = "IMAGE" }
      ]
      ```

      The request is successfully processed with the remaining button `2` and the following response is  provided to mobile application:
      `success=true, resultCode = "WARNINGS", info="Invalid enums were removed: softButtons[1].type:UNKNOWN"`

## Potential downsides

This proposal does not solve versioning issues. For example:
In the MOBILE API exists the "DisplayCapabilities" parameter which is deprecated since the 6.0 API version. In the case when a Mobile Application has the 6.0 API version and the SDL Core has the 5.0 version and Mobile Application sends a message to the SDL Core without the "DisplayCapabilities" parameter, it brought the SDL Core to a core crash.

To avoid a similar situation the SDL Core should have the mechanism to negotiate his own API version with Mobile Application API versions and choose the correct way to process messages. The mechanism is out of the scope of this proposal and should be described in a separate proposal.

## Impact on existing code

Extend SDL Core logic by additional functionality of unknown values filtering.

## Alternatives considered

The authors were unable to determine any alternative solutions.