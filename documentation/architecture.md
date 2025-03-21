# Architecture of the project

This document contains details of the architecture of a11ysupport.io, including

* grading method
* file structure
* data model
* build process
* front end
* workflow to update data

## Grading method and high level overview

Note: I'm not entirely happy with this grading method and structure. Issues include:

* Quality of support is not indicated (for example, if a feature is minimally supported, potentially misleading, or goes above and beyond).
* Bubbling support to the feature level can obfuscate important issues with support.
* Support terminology isn't always consistent (a result of my rapid prototyping).
* JSON structure was optimized for making it easy to build a front end, not necessarily for maintenance, outside consumption, or editing by contributors. For example, an array (instead of an object) for results might make more sense and would be easier to add AT in the future. I see this as a minor issue.
* We don't track the screen reader mode (but it is implied via the commands)

Details of the grading method and a high level overview of the project include:

* AT/browser combinations are divided into `core` and `extended` categories and support is tracked independently for each category.
    * Core combinations are kept to a minimum and include the most common combinations as found by surveys and other research.
    * Extended combinations include any combination of AT/browser that might exist on a given OS.
* Only assertions where `assertion.type === "MUST"` contribute to `core` support.
* Support values are defined in an `output.result` by a contributor (either by editing the JSON directly or by filling out a form in the front end). Possible values of `output.result` include `pass`, `fail`, or `partial`.
* `output.result` indicates the result of an assertion of a specific feature applied to a specific test, applied to a specific input command for a specific AT/browser combination. It is the most specific result in the project. For example, the following might reference the result of a test against NVDA/Firefox: `test.assertions[0].results.nvda.browsers.firefox.output[0].result`
* A single AT/browser combination can have many output results.
* During the build process, output results are bubbled to:
    * The browser object `test.assertions[0].results.nvda.browsers.firefox.support`. Values are mapped to one of `y` (support='yes'), `n` (support='no'), or `p` (support='partial').
        * If all output results were `pass`, then output maps to `y`.
        * If all output results were `partial`, then output maps to `p`.
        * If no output results passed, then output maps to `n`.
        * If there was a mix of values, map to `parital`.
    * The AT object `test.assertions[0].results.nvda.core_support` and `test.assertions[0].results.nvda.extended_support` (which includes unique support values for all browsers)
    * The test assertion `test.assertions[0].core_support` and `test.assertions[0].extended_support`, 
    * The test object `test.core_support` and `test.extended_support` which include unique support values for all at
    * The referenced feature assertion object `feature.assertions[0].core_support` and `feature.assertions[0].extended_support`

## file structure

From the root directory of the project

* `/build/` contains the JSON files that are generated during the build process and used by the front end
* `/data/` contains the JSON data that drives the project
  * `/data/schema/` contains JSON files that describe other JSON files (used to validate JSON files)
  * `/data/tech/` contains JSON files that describe different technology features
  * `/data/tests/` contains JSON files that describe tests and test results
    * `/data/tests/html/` contains the static HTML files referenced by tests
* `/documentation/` contains documentation on the project
* `/public/` public facing assets that are served by the front end
* `/routes/` contains logic to route front end requests
* `/scripts/` contains scripts to manage the project
* `/src/` contains helper functions and other source files
* `/views/` contains PUG files for the front end

## Data model

This section describes the data model

### Tech data model

The project supports tracking the support of many different technologies and their features, such as HTML, SVG, ARIA, CSS, etc.

A master list of the technologies can be found at [/data/tech.json](https://github.com/accessibilitysupported/a11ysupport.io/blob/master/data/tech.json)

Within the `/data/tech` directory are subfolders for each tech ID found in `/data/tech.json`. For example, `aria` would map to `tech.aria` and `/data/tech/aria/`. Each file within `/data/tech/aria/` describes a specific technology feature (attribute, role, element, etc).

#### Tech features

Taking `aria-haspopup` as an example, the associated technology object is located at [/data/tech/aria/aria-has-popup_attribute.json](https://github.com/accessibilitysupported/a11ysupport.io/blob/master/data/tech/aria/aria-haspopup_attribute.json). The trailing '_attribute' part of the name is not necessary, but helps track the type of feature when browsing the folder.

The following properties can be provided by a contributor:

* `title` (string|required): a human readable title for the feature.
* `type` (string|optional): the type of feature. Usually one of `element`, `role`, or `attribute`.
* `description` (string|required): a human readable description of the feature in markdown format.
* `related_issues` (array|optional): an array of url objects that point to related issues in browsers or assistvie technologies.
* `references` (array|optional): an array of url objects that point to specifications or other resources.
* `reccomendation` (string|optional): a markdown string that descibes any reccomendations for authors.
* `date_updated` (string|required): the date that this feature object was last updated.
* `keywords` (array|optional): array of strings used to help surface search results
* `assertions` (array|optional): an array of assertion objects that describe what is required for a given feature to be "supported"

Properties that are generated during the build process include:

* `id` (string|built): this is not required and must match the file name. If it is not defined here, the build process will define it using the file name. The ID is prefixed with the tech. `aria-haspopup` would be `aria/aria-haspopup_attribute`.
* `keywords_string` (string|built): a string that is created during the build process by imploding the `keywords` array.
* `tests` (array|built): the build process will be created during the build process and contains an array of associated test objects. This makes it easier to build the front end.
* `core_support` (array|built): an array of strings that describe unique support values found in tests
* `core_support_string` (string|built): an human readable string that describes the support based on the values found in `core_support`
* `core_support_by_at` (object|built): an object that describes the core support for each AT. Just another way to describe support.
* `core_support_by_at_browser` (object|built): an object that describes core support for different core at/browser combinations
* `extended_support` (array|built): an array of strings that describe unique support values found in tests
* `extended_support_string` (string|built): an human readable string that describes the support based on the values found in `extended_support`

#### Tech feature assertion object

The assertion object located at `feature.assertions[0]` describes an assertion that must be met for the feature to be considered as supported.

The following properties can be provided by a contributor:

* `id` (string|required): this is a human readable ID for the assertion. Spaces are not allowed, just alphanumeric with underscores.
* `title` (string|required): the assertion text itself. This MUST contain one of "MUST", "SHOULD" or "MAY", and may be written to be conditional.
* `type` (enum|required): one of "MUST", "SHOULD", or "MAY". "MUST" assertions are considered the baseline for minimal support and are the only type of assertion that contributes to 'core' support.
* `css_target` (string|required): the css selector for elements to test this assertion against. May be overridden by a test.
* `preconditions` (array of string|optional): an array of strings that describe the preconditions for this assertion to be applicable
* `operation_modes` (array of strings|required): contains command tags. Usually one of `sr/interaction`, `sr/reading`, or `vc`. These tags control which commands are shown to users while testing and to determine which types of AT the assertion applies to.

Properties that are generated during the build process include:

* `tests` (array|built): an array of test objects that reference this assertion
* `supports_at` (array|built): an array of `at.type` values that this assertion supports (determined by `test.operation_modes`)
* `core_support` (array|built): an array of strings that describe unique support values found in tests
* `core_support_by_at_browser` (object|built): an object that describes core support for different core at/browser combinations
* `extended_support` (array|built): an array of strings that describe unique support values found in tests

### Test data model

Continuing with the `aria-haspopup` example, a single feature might have many tests. A test for `aria/aria-haspopup_attribute` is the file [/data/tests/tech/aria/aria-haspopup_listbox.json](https://github.com/accessibilitysupported/a11ysupport.io/blob/master/data/tests/tech/aria/aria_haspopup_listbox.json). There are other tests for other attribute values. The support result of all of these tests bubbles to a single value of support for the feature itself.

The following properties can be provided by a contributor:

* `type` (enum|required): the type of test (`custom` or `assertion`)
* `title` (string|required): the title of the test.
* `description` (string|required): a longer markdown formatted description of the test.
* `html_file` (string|optional): the path, relative to the `data/tests/` directory for the HTML test file. If empty, the same path as the test file is assumed (relative to the `data/tests/` directory)
* `assertions` (object|required): an object that describes the assertion that must be met for the test to pass.
* `history` (array|required): an array of history objects that describe how the test and results have changed over time.

Properties that are generated during the build process include:

* `id` (string|built): the ID of the test, based on the file name.
* `core_support` (array|built): an array of strings that describe unique support values found in tests
* `core_support_string` (string|built): an human readable string that describes the support based on the values found in `core_support`
* `core_support_by_at_browser` (object|built): an object that describes core support for different core at/browser combinations
* `extended_support` (array|built): an array of strings that describe unique support values found in the `results` object
* `extended_support_string` (string|built): an human readable string that describes the support based on the values found in `extended_support`

#### The test `assertion` object

The test assertion object creates a link to a feature assertion object.

The following properties can be provided by a contributor:

* `feature_id` (string|required): the ID of linked feature
* `feature_assertion_id` (string|required): the ID of the feature object on the linked feature
* `css_target` (string|optional): a CSS target that overrides the more generic one provided by the feature assertion (this can be specific to the test)
* `expected_value` (string|optional): The expected value of the assertion test result (can be helpful in some circumstances)
* `results` (object|required): an object that describes the testing results

Properties that are generated during the build process include:

* `supports_at` (array|built): an array of `at.type` values that this assertion supports (determined by `test.operation_modes`)
* `feature_title` (string|built): the title of the linked feature
* `assertion_title` (string|built): the title of the linked feature
* `operation_modes` (array|built): an array of command tags that describe operation modes
* `core_support` (array|built): an array of strings that describe unique support values found in tests
* `core_support_string` (string|built): an human readable string that describes the support based on the values found in `core_support`
* `core_support_by_at_browser` (object|built): an object that describes core support for different core at/browser combinations
* `extended_support` (array|built): an array of strings that describe unique support values found in the `results` object
* `extended_support_string` (string|built): an human readable string that describes the support based on the values found in `extended_support`

### `results` object

The `results` object (located at `test.assertions[0].results`) can contain a property for each of the at `at` objects listed in [/data/ATBrowsers.json](https://github.com/accessibilitysupported/a11ysupport.io/blob/master/data/ATBrowsers.json). For example, support data for `nvda` would live under `test.assertions[0].results.nvda`.

Each `results` object chan contain a property for each of the browsers listed in [/data/ATBrowsers.json](https://github.com/accessibilitysupported/a11ysupport.io/blob/master/data/ATBrowsers.json). These property names map to the ids of AT defined in `ATBrowsers.json`. For example, support data for `nvda` and `firefox` would live under `test.assertions[0].results.nvda.browsers.firefox`. These objects are known as `browser` objects.

All `at` and `browser` combinations are optional, and any gaps will be filled in by the build process with empty data (unknown results). This makes iterating over the built data easy.

#### the `at` object

The following properties can be provided by a contributor:

* `browsers` (object|required): an object with properties that map to each supported browser for the AT

Properties that are generated during the build process include:

* `id` (string|built): the ID of the AT as found in `ATBrowsers.json`
* `core_support` (array|built): an array of unique support values of core testing combinations from its children
* `core_support_string` (string|built): an string that describes `core_support`
* `extended_support`: (array|built): an array of unique support values of extended testing combinations from its children
* `core_support_string` (string|built): an string that describes `extended_support`

#### the `browser` object

Each `browser` object then contains the following properties which can be provided by the contributor.

* `at_version` (string|required): the version of the AT used during the test
* `browser_version` (string|required): the version of the browser used during the test
* `os_version` (string|required): the version of the OS used during the test. The OS name can be inferred.
* `output` (array|required): an array of output objects that describes the specific output and results for different interactions.
* `date` (string|required): the date that this at/browser combination was last tested.
* `notes` (string|optional): any notes that describe findings or jusitfy the result.

Properties that are generated during the build process include:

* `support`: (string|built) the combination of support values for the output array y=yes, p=partial, n=no, na=not-applicable.

#### the `output` object

The `output` object contains the following properties which can be provided by the contributor:

* `command` (string|required): The ID of the command used to navigate or trigger the element that matches the css target. These IDs match those found in the [/data/ATBrowsers.json](https://github.com/accessibilitysupported/a11ysupport.io/blob/master/data/ATBrowsers.json) array of commands for the current AT.
* `output` (string|required): the output of the result.
* `result` (enum|required): One of `pass`, `fail`, or `partial`.

## Build process

The build process:

* loops over each feature, test, and assertion, combining them and adding empty at/browser combinations where none have been defined
* bubbles support data from the the `output` object all the way to the `feature` object, creating matching support strings along the way
* outputs all of this generated data to the `/build/` directory.

This built data makes coding the frontend easy. The frontend itself does not contain any logic around filling in gaps of support data or bubbling information.

## Front end

The frontend is built with NodeJS, Express.js, Pug templating, JavaScript, and CSS.

## Workflow to update data

1. User searches for feature
2. User selects a test
3. user selects 'run test'
4. User selects their testing combination from a select list
5. User sees testing instructions and previous results
6. User performs the tests
7. User logs findings to the form (pre-filled with previous data)
8. User submits the form and is taken to github with an issue body already filled out. The issue body contains a table that describes their results.
9. User submits github issue
10. Discussions happen if necessary. Findings are verified.
11. admin uses the script at `/scripts/sync-support-point.js` by giving it a GitHub issue ID. The script finds the table in the issue and updates the JSON file accordingly.
12. admin commits changes and pushes.
13. results are eventually deployed to production

