# API Checker

A tool to check if an API is working as expected from the point of
view of the API client. Allows to check a sequence of related calls and
detects any issue happening in the intermediate stages, providing an
exact report of the found issue.

A main [YAML](https://yaml.org) input file describes the behaviour
of the tool during execution. Such behaviour is defined as a list
of consecutive stages, each comprised of a request and its related
response validation. Every stage can be the input for any of the next
stages. If an stage execution does not match the expectations, API checker
stops and reports an adequate error. When all the stages match the defined
expected values, the tool outputs to a file, to console or to a network
service.

Other YAML input files can be also provided, defining environment
parameters, user profile data or any additional configuration that could
be needed for the tests. All them complete an API checker single
test case execution.

In case that several files describing the tool behaviour are given
as input, the tool is able to prepare multiple test cases and launch
them sequentially, reporting every case results and a final summary
of the testing process run.

![Architecture](apichk.png)


# Installation

You can download API Checker last release tarball for your platform
(Win64 or Linux_x86_64 supported) [here](https://github.com/telefonicaid/apichk/releases/latest) under the name of: 

| OS | ditribution file |
| -- | ---------------- |
| Windows | apichk_[VERSION]_win_[ARCH].zip |
| Linux | apichk_[VERSION]_linux_[ARCH].tar.gz |

Where VERSION is the identifier of the release and ARCH can be `amd64` or `X86_64`

Uncompress the file somewhere in your local machine. An _apichk_ subfolder
will be created. Move to such folder and you should find there the 
**apichk** binary and the **README.md** file with the instructions to run it.


# Run the tool

You can run the tool by invoking the executable *apichk* along with some positional
params which represent the different input yaml files to use for testing. You can 
explicitly reference such files or the folder containing them.

> Note:
>    The code snippets onwards refer to Unix environment. Windows analog executions involve
>    corrections in the path symbols (backslash) but they are basically identical


## Run with one positional parameter

In case that only one positional parameter is provided, a deep search of yaml files 
is performed:

- If the parameter matches a subfolder: Any yaml file found into that folder and
    subfolders is considered for testing.

- If the parameter matches a yaml file: That file and any other yaml file into
    subfolders is considered for testing.

The tool gathers the longest n-ary cartesian product of the files in a folder with the
ones in subfolders and parents to conform a proper list of tests cases. After that,
it executes all the cases displaying a summary of the results at the end or stopping
when a first error happens (see the -c optional parameter if wanted to continue till
the end in case of error)
If any of the tests lacks the stating spec file or has more than one of that type
the test fails.

### One positional parameter examples

#### Example 1

If we have a folders structure like:

```
-myfolder
    a.yaml
    -mysubfolder
        b.yaml
        c.yaml
        -anotherfolder
            d.yaml
            e.yaml
        -anotheronefolder
            f.yaml
```

and I execute 
```
apichk myfolder
```

This equivalent execution is generated
```
apichk a.yaml b.yaml d.yaml
apichk a.yaml b.yaml e.yaml
apichk a.yaml b.yaml f.yaml
apichk a.yaml c.yaml d.yaml
apichk a.yaml c.yaml e.yaml
apichk a.yaml c.yaml f.yaml
```

but the particularity that all them are executed at a time, reporting a summary at 
the end of the execution.

#### Example 2

Another example:

If we take the same folders structure as in example 1, but we execute
```
apichk myfolder/mysubfolder/b.yaml
```

The generated tests are
```
apichk a.yaml b.yaml d.yaml
apichk a.yaml b.yaml e.yaml
apichk a.yaml b.yaml f.yaml
```

As you can see, the _c.yaml_ file is not considered. In general this applies to
any folder at the same level and its subfolders

#### Example 3

Let's take again the same structure of files and folders of the example 1
and let's execute:
```
apichk myfolder/mysubfolder/anotheronefolder
```

The generated tests are
```
apichk a.yaml b.yaml f.yaml
apichk a.yaml c.yaml f.yaml
```

## Run with several positional parameters

When provided more than one yaml file, the tool executes only one test case with 
all the argument yaml files. It is only mandatory to have one and only one stages spec
yaml file. Any other number of yaml files with the additional environment, profile 
or whatever configuration can be given.

Mandatory parameters:
* stages spec: at least one and only one yaml file with the test case behaviour

Optional parameters:
* configs: additional yaml files with configuration required by the test like
    environment settings or a user profile

i.e.
```
apichk es_pwd_stages.yaml es_pro_env.yaml es_pro_profile_51070230H.yaml
```
executes the tool taking the three yaml files as input. Internally, it'll 
detect _es_pwd_stages.yaml_ file as stages specification and _es_pro_profile_51070230H.yaml_ 
and _es_pro_env.yaml_ as additional configuration files

## Optional parameters

Some optional parameters can be indicated along with the positional ones:

* -h, --help: shows the help of the tool
* -v, --verbose: provides more log details, including the raw HTML messages. You can
    set more than one parameters of this type. If set at least two of them (-vv)
    the raw HTTP messages include the whole HTML content, otherwise the HTML content
    is replaced by a phrase indicating that its HTML content has been stripped. Other
    type of contents are never stripped.
* -q, --quiet: reduces the log details of the test by only printing the input and
    the output but skipping any intermediate progress, except in case an error
    happens.
* -l, --logs: path where logs are created and filled. Default is set to the system
    temporary folder (i.e. /tmp for unix like systems)
* -c, --continue-on-error: if an error happens during execution, when this flag
    is set, the execution continues till the end.

## Optional parameters examples

```
apichk ob -v
# prints out to /tmp/apichk_msgs.log the raw HTTP messages
```

```
apichk ob -vv
# prints out to /tmp/apichk_msgs.log the raw HTTP messages, including HTML content
```

```
apichk ob -q
# does not write to /tmp/apichk_msgs.log
# does not print to stdout the intermediate process of every test case
# only prints input and ouput data for each test
```

```
apichk ob -l /a/path -c
# prints log files to /a/path
# continues to the next test in case an error happens
# at the end of the complete execution, a summary of the results is provided
```


# Define the behaviour

In this section are covered the details to configure the tool for executing
the exact desired test.

## Input data

Every test executed by the tool requires at least one mandatory input
file where test behaviour is defined. We call it onwards the _spec_.
Additional files with the environment or user profile configuration could
also be needed for a proper execution. All those files provide
the data to be consumed in runtime by the tool.

### Stages Spec

The mandatory input for the tool is a yaml file defining the sequence of
requests needed to accomplish the testing process and the results to
dump out to console.
It is mainly comprissed of a list of stages. every stage details a request
details to send and the expected response. 
The elements defined for the stage request portion can be literals
or placeholders which are replaced by their related value before
the send.
The content of the stage response includes the data whose presence
is expected and optionally its value or a regular expression that
should match its value.
The other main element in the spec is the dump section. It defines
which elements of the executed stages you want to show in console.

An spec example file could look like this:

```
---
stages:
    - get-form:
        request:
        method: GET
        url: "{{ config.get-form-url }}"
        response:
        status-code: 302
        headers:
            location:
    - post-msisdn:
        request:
        method: POST
        url: "{{ get-form.response.headers.location }}"
        headers:
            accept: "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8"
        body:
            user: "{{ config.name }}"
            password. "{{ config.password }}"
        response:
        status-code: 200
dump:
    final_body: "{{ post-msisdn.body }}
```

### Environment

The second common input for every test case is a yaml file defining the details of 
the environment where testing. This includes specific URIs, static 
redirection URLs, authentication endpoints, machine access credentials,
etc..

An example of such file would be:

```
---
authorize-url: https://cert-machine.com/authorize
client-id: my-client
client-secret: my-secret
redirect-uri: https://redirect/url
token-url: https://cert-machine.com/give/me/a/token
```

### Profile

The third common input for the tools uses to be another file defining the details
of the user/system/actor sending the requests defined in the spec. It could
include username, password, mobile phone, specific requested data related to
such user, etc..

An example of such data would be:
```
---
username: myuser
password: mypassword
msisdn: 666777666
```

## Stages

The stages are the steps to carry out the test. Every step matches one specific
request plus the data send plus the validations to make over the response.
The stages are defined into the spec file. The content of such file is the list,
in order, of all them. The tool launches one by one every related request and
will only stop if a definition error is found or if a validation in a stage
is not passed, reporting the related error result.

Every stage must have a mandatory name, a definition of the elements of the 
request to send, and a definition of the expected response elements and their
values.

### Name

The name of the stage is the way that the tool has to find a value
into such stage if it is needed from another future stage. It is, in fact,
how to refer something from the stage in any case. It must be different
from any other name of any other stage into the spec file.

```
---
stages:
    - my-first-stage:
        request:
            ...
        response:
            ...
    - my-second-stage:
        request:
            ...
        response:
            ...
    ...
```

### Request

Every stage requires a request element with the data to send in a request.
Such values can be set by literals, by referring any configuration param taken
from the environment or profile, or by referring a value got in a 
previous stage execution.

The mandatory elements in a request are
* method: The method of the HTTP request
* url: The URL where sending the HTTP request

The optional elements in a request are:
* redirect: boolean flag set to true if wanted the stage response to be redirected
    in case a 3xx response code is received. Default is false.
* headers: map of headers and their values to send
* params: map of query params and their values to send
* body: map of elements to send.

Any of the values of method, url, headers, params or body, you can define 
literals or placeholders. In the case of the maps, even the keys can be
placeholders that will be replaced by their real value before sending the request

```
---
stages:
...
    my-stage-name:
        request:
            redirect: false
            url: http://theendpoint.com
            method: POST
            headers:
                content-type: application/json
                accept: application/json
                x-my-custom-header: my-custom-value
            params:
                my-query-param: my-query-param-value
                name: value
            body:
                name1: value1
                name2: value2
            ...
```

### Response

Every stage requires a response definition. In this case the defined values are
the ones expected to be included into the got response. 

The mandatory elements in a response are:
* status-code: You can include here a literal expected status code for this response
    or a regular expression matching several.

The optional elements in a response are:
* headers: map of the headers that we expect will be included in the response. 
    If a value or regular expression is provided for a header, it will be validated
    that the got response includes a header value matching it
* body: the expected body content. You can define a map of values that will
    be evaluated separatedly, taking the body content as a json map. This way
    you can include individual validations for every element into the body json content
* swagger: an URL to a public endpoint of a swagger specification to validate the
    response against. This parameter can also be a local path into the machine where
    the tool is executed

```
---
stages:
...
    my-stage-name:
        request:
            ...
        response:
            status-code: 201
            headers:
                content-type: application/json
            body:
                name1: value1
                name2: value2
            swagger: https://docs.baikalplatform.com/user_profile/v3.3/user_profile.json
            ...
```
## Loop

In case there is a need for an stage to be repeated until some specific response is 
received, we can define a loop stage. The name of the stage is not relevant but it is
the content. Instead of defining request and response elements, we can define the
maximum times to iterate, the step sleep in seconds and the condition to leave the loop.

The mandatory elements of a loop are:
* condition: Placeholder with a boolean expression to be evaluated at the end of every loop step.
    If the expression returns true, the loop is finished. Otherwise, a new step with the
    same stages wrapped into the loop is executed again.
* stages: List of stages to be executed in every loop step. The syntax for these stages
    is the same as in the case of the general list of stages.

The optional elements of a loop are:
* max-retries: Max number of loop steps to be executed before exiting with error
* sleep: Time in seconds to wait between loop steps.

```
- apimsgr-login-status-polling:
    stages:
    - apimsgr-login-status:
        request:
            method: POST
            url: "{{ config.login-status-url }}"
            headers:
            Content-Type: multipart/form-data
            body:
            request:
                jsonrpc: 2.0
                id: 2
                method: 'Login.3.status'
                params:
                sessionToken: "{{ apimsgr-login-request.response.body.result.sessionToken }}"
            x-tuenti-apicontext:
                installationId: "{{ config.installationId }}"
                deviceFamily: wbteNCLH4mncE2ffKH35wvWlAEHIuWUTT8EfQu5K
                screenSize: xhdpi
        response:
            status-code: 200
    condition: "{{ $equals( apimsgr-login-status.response.body.result.nextStep, 'finished' ) }}"
    max-retries: 10
    sleep: 2
```

In every loop step, the involved stages are reset to their original values.

## Additional Sections

Next sections can be defined in any stages spec file. Some will take effect at the
end of every test case and others will do it at the end of the complete execution
of the tool.

### Dump

After the stages section of the stages spec file you can find the dump one. There
you can define which elements (headers, bodies, query parameters, etc...) you want
to dump out to the console once the execution has been completed succesfully. 
This is an interesting way of showing up interesting data as result of the testing
process.

You can use placeholders as in stages to resolve references to data got in previous steps:

```
dump:
    access_token: "{{ token.response.body.access_token }}"
    id_token: "{{ token.response.body.id_token }}"
    scopes: "{{ $pretty($as_list(token.response.body.scope)) }}"
    decripted_access_token: "{{ $pretty($jwt_decode(token.response.body.access_token)) }}"
    decripted_id_token: "{{ $pretty($jwt_decode(token.response.body.id_token)) }}"
```

> Note:
>   Dump section has effect only on single test cases. When tool executes more than one
>   test case it is ignored

### Publish

This section defines one or several external systems which will receive the defined results
of successful executions. You can define a network endpoint or a file where writting out 
some of the resultant values of the testing process. There is a limited set of publishers
that you can use for this purpose.

In case that the execution comprises more than just only one test, the values defined
for this section are updated at the end of every case execution. This allows to publish
variables that are specific to every test.

Once the tool ends all the cases, it publish what has been saved during the execution

#### Postman publisher

In order to publish the results with this publisher it is needed that you sign in with 
Postman that will create a cloud profile with the environments and collections that 
you use into the native application. You'll be requested to do it the first time
you enter into the installed application or you can do it here:

```
https://app.getpostman.com/signup?redirect=web
```

Once registered, it is needed to access the dashboard where you can request an API key
that is needed to update any cloud data from the tester script:

```
https://go.postman.co/integrations/services/pm_pro_api
```

Click on the button *Get API Key* to create a new API key. In the section *Existing API Keys*
you can obtain the list of the active ones

Now it's time to add a configuration to the stages spec file with the definition for
publishing the results to your Postman account. Add a new _publish_ section, like this:

```
---
stages:
...
dump:
...
publish:
  postman:
    url:  https://api.getpostman.com
    api-key: yiulf3a81521784bd9b57291099169a8b9
    environment: my-testing-environment
    variables:
      Token: "{{ token.response.body.access_token }}"
      RefreshToken: "{{ token.response.body.refresh_token }}"
```

The mandatory values are:
* url: the url of the postman server. By default it is 'https://api.getpostman.com'
* api-key: the value obtained above for accessing to your cloud collections and environments
* environment: the environment that you want to update with the values got from the script
* variables: a map of key-value pairs with the parameters that you want to update in the 
    environment stored in the server. Only the ones declared here will be updated. The 
    rest of the existing values will not be updated at all. You can refer as placeholders
    like in any other stage


An even more elegant way of declaring the publish section by using placeholders to a 
configuration file. This way, we can have an config definition where setting the
api-key, environment and postman url which will be referred from the publish section:

```
# my config file.yaml
publish:
  postman:
    url: https://api.getpostman.com
    api-key: yiulf3a81521784bd9b57291099169a8b9
    environment: my-testing-environment

# ny spec file.yaml
publish:
  postman:
    url: "{{ config.publish.postman.url }}"
    api-key: "{{ config.publish.postman.api-key }}"
    environment: "{{ config.publish.postman.environment }}"
    variables:
      Token: "{{ token.response.body.access_token }}"
      RefreshToken: "{{ token.response.body.refresh_token }}"
```

##### Multiple test publish

When executing several tests at once, it is possible to use placeholders to gather
different variables for the different cases into the publisher. Once everything
finishes, the publisher will send to the configured external systems the saved
variables.

Let's focus on the previous example and modify the name of the variables this way:
```
...
publish:
  postman:
    url: "{{ config.publish.postman.url }}"
    api-key: "{{ config.publish.postman.api-key }}"
    environment: "{{ config.publish.postman.environment }}"
    variables:
      Token: "{{ token.response.body.access_token }}"
      RefreshToken: "{{ token.response.body.refresh_token }}"
      "{{ $concat('Token[', config.username,']') }}": "{{ token.response.body.access_token }}"
      "{{ $concat('RefreshToken[', config.username,']') }}": "{{ token.response.body.refresh_token }}"
```

Now the var names depend on the `config.username` so that for every new value
on that specific config, there will be new published variables like:

```
Token[+34666888888]
RefreshToken[+34666888888]
Token[+34666777777]
RefreshToken[+34666777777]
...
etc
...
```

## Multiple test output

When apichk detects that more than one test must be executed, it skips the dump
and publish sections of the stages specs and instead, cummulates numerical results
of every case execution. At the end of all tests run shows up a summary of the 
whole execution with the succeeded, failed and total number of tests.
Just below that summary, in case any of them fails, a list of individual tests
input is shown, for better identification of problematic cases.

```
******************
Executed:
        Cases:     4
        Succeeded: 3
        Failed:    1
******************

******************
Failed Cases:
        pro/es_pro_env.yaml
        pro/pwd/es_login_pwd_stages.yaml
        pro/pwd/profiles/es_pro_profile_510707780T.yaml
------------------------------------------------------
```

Remember that -c (--continue-on-error) modifier is needed to execute the whole
battery of tests. If not provided that modifier, the execution stops on first error

# Stages Spec References And Functions

Into the stages spec it is crucial having a way to reference, from a specific stage,
configuration parameters, values of previous stages or calculate new values by
manipulating one or several existing ones by applying some functions. Here is
where references (aka placeholders) and functions make sense

## Placeholders

Sometimes we cannot determinate the exact value of a parameter, body, header, etc...
when designing the spec because it is calculated in a previous stage or belongs
to a profile or environment configuration. This is particularly useful in the dump
section where the values to show are not known until certain previous stages
have been completed.
In that situation, there is a way to set a placeholder instead of a literal. You must
sorround certain reference between quotes and curly brackets, like this

```
"{{ my.reference }}"
```

That reference can be a path to a config, a previous stage value, a function or 
a group of nested elements comprising any of them

### Config References

References to values defined in environment or profile yaml files start with the _config_ word
plus a dot and the name of the parameter into the original file. This way, a config
parameter set into, say, the profile yaml file like

```
username: frederico
```

is referred into the spec like
```
"{{ config.frederico }}"
```

regardless if it is included into the profile or the environment file, because in both
cases the taken reference is a configuration.

### Previous Stages References

If a value to use in a stage is not known until certain previous stage has been 
completed, you can refer to it by the name of that stage. Say that we need to
take the value of a json parameter _server-url_ included in a previous response
body, to be the url to send the next one. Here is the solution

```
---
stages:
...
    - previous-stage:
        request:
            ...
        response:
            body:
                server-url:
    - current-stage:
        request:
            url: "{{ previous-stage.response.body.server-url }}"
            ...
    ...
```

We are determinating two things here; the fist is validating the existence of a 
body key in _previous-stage_ named *server-url*. And the second is that such
value is going to be used in _current-stage_ as request URL

> Note:
>   The value referred in a placeholder can be in any previous stage, not the just
>   inmediatly before

## Functions

If wanted to take a value from a previous stage, but partially. Or if the value is 
included in a huge html content. Or if it is needed to process somehow a value, the 
tool includes a buch of functions to handle this. 
Such functions must be referred with the *$* symbol precedding its name and 
enclosing its parameters between *(*  and *)*. The parameters can be other
functions, literals or placeholders:

```
"{{ $thefunction('param_one', previous-stage.param.two) }}"
```

Let's see the existing functions:

### as_list

Takes an string content and returns a list of tokens by splitting it by a separator.

mandatory params:
* content: the string to parse to a list

optional params:
* separator: the character to divide the resultant tokens. Default is blank space

i.e.
```
"{{ $as_list('a random string') }}"
```
returns _['a', 'random', 'string']_

### b4

Encodes a string as base64

param:
* content: the string to be base64 encoded
i.e.
```
"{{ $b4('never gonna give you up, never gonna say goodbye') }}"
```
returns _bmV2ZXIgZ29ubmEgZ2l2ZSB5b3UgdXAsIG5ldmVyIGdvbm5hIHNheSBnb29kYnll_

### b4_decode

Decodes a base64 string

param:
* base64 string: the content to be decoded
i.e.
```
"{{ $b64_decode('bmV2ZXIgZ29ubmEgZ2l2ZSB5b3UgdXAsIG5ldmVyIGdvbm5hIHNheSBnb29kYnll') }}"
```
returns _'never gonna give you up, never gonna say goodbye_

### concat

Concats several values. If they conform a path, duplicate slashes
are removed

params:
* list of elements
i.e.
```
"{{ $concat('one', '/last/', '/question') }}"
```
returns _one/last/question_


### decode_saml
Decodes and inflates a saml content

params:
* content: the string zlib and base64 encoded
i.e.
```
"{{ $decode_saml('jVJdb9owFP0rkd8TJ2F8xEpSsXXVKhUNAWXSXibjXIq32Ka+Tmj//RwStPKC9np9z4fPufndm6qDFixKowuSRDEJQAtTSf1SkOfNQzgjd2WOXNXpkc0bd9AreG0AXeCBGln/UpDGamY4SmSaK0DmBFvPF08sjWJ2tMYZYWoSzBHBOi/1xWhsFNg12FYKeF49FeTg3BEZpdyrRCDCo4Vox+UfXh9r7vbGqkgYRTvFcL3+ToJ7b0Nq7s7WL+hT2+GUaSU6bjuE56ILuRgmv35sKQkejBVw/k5B9rxGIMHjfUE4TCbZy++d2EFWTaej6X4EM7nPKphUU7+CS44oW/gHQmzgUXti7QqSxkkWxmmYJJskZaOMJUk0G49/kmA5RPBZ6j7aW3nt+iVk3zabZbiCSloQjgTbS01+iQylsLMD+7GN2+T8UgEp/zPwnH5UKnNVsa9vDnTnBQdhVd1WVeB4xR33ti28srmwW177MxrgftY32BV4OkXtFEE0Vrr3vnJxAMWpj3dM40/U9hdIylFOr9jKnF6ZKwfr15db/gU=') }}"
```
returns
```
<?xml version="1.0" encoding="UTF-8"?><saml2p:AuthnRequest xmlns:saml2p="urn:oasis:names:tc:SAML:2.0:protocol" AssertionConsumerServiceURL="https://auth.ec-pre.baikalplatform.com/saml-SSO" Destination="https://wvpre.movistar.com.ec/MiMovistar_WV/" ForceAuthn="false" ID="ae669gjbcbe9d7737f3e8if9de6d7" IsPassive="false" IssueInstant="2019-02-11T12:39:11.855Z" ProtocolBinding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect" Version="2.0"><saml2:Issuer xmlns:saml2="urn:oasis:names:tc:SAML:2.0:assertion">https://auth.ec-pre.baikalplatform.com/</saml2:Issuer><md:Extensions xmlns:md="urn:oasis:names:tc:SAML:2.0:metadata"><req:AcrValues xmlns:req="http://www.v7security.com/schema/2015/04/request">3</req:AcrValues></md:Extensions></saml2p:AuthnRequest>
```

### equals

Returns true if incoming parameters are equals

params:
* value1: first value to evaluate
* value2: second value to evaluate
* valueN: N value to evaluate
i.e:
```
"{{ $equals('what', 'what', 'what') }}"
```
returns _true_ boolean value

### fragment

Returns the fragment part of an URL. If the URL does not have fragment, returns empty value

params:
* url: from where extracting the fragment
i.e:
```
"{{ $fragment('http://my.url.com#myfragment?lalala=lololo') }}"
```
returns _myfragment?lalala=lololo_

### hex_decode

Decodes an string of hexadecimal values to ascii

params:
* hex content: the string of hex characters
i.e.
```
"{{ $hex_decode('6e6576657220676f6e6e61206769766520796f752075702c206e6576657220676f6e6e61206c657420796f7520646f776e') }}"
```
returns _never gonna give you up, never gonna let you down_

### host

Returns the host of an URL, including the port

params:
* url: from where extracting the host
i.e.
```
"{{ $host('https://my.url.com/path') }}"
```
returns _my.url.com:443_

### hostname

Returns the host of an URL (not including the port)

params:
* url: from where extracting the hostname
i.e.
```
"{{ $hostname('https://my.url.com/path') }}"
```
returns _my.url.com_

### json_extract

Returns the value for a specific key

params:
* key: the key to search for into the JSON. A placeholder of dot separated keys 
    is also valid if wanted to extract a value not in the first level of the map
* content: the JSON with the data. It can be a dict or an string that is JSON parseable
i.e.
```
"{{ $json_extract('thekey', '{"a":"value", "thekey": "thevalue"}) }}"
```
returns _thevalue_

### jwt_decode

Decodes a [JWT](https://jwt.io) token and returs its content

params:
* token: the JWT token to decode
i.e.
```
"{{ $jwt_decode('eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c') }}"
```
returns _{"sub": "1234567890", "name": "John Doe", "iat": 1516239022}_

### jwt_encode

Encodes a payload as [JWT](https://jwt.io) token

mandatory parameters:
* key: the secret key used to encode the token
* payload: the JSON content of the token

optional parameters:
* iat: if wanted to include the time that the token has been issued [iat](https://tools.ietf.org/html/rfc7519#section-4.1.6) with the current unix time.
* exp: if wanted to include the expiration time [exp](https://tools.ietf.org/html/rfc7519#section-4.1.4). It must be assigned to the difference time to take with the iat
* headers: a JSON dictionary with additional headers to be included into the token
* algorithm: the algorithm to use for the token encryption. Default is 'HS256'

i.e.
```
"{{ $jwt_encode('my_key', '{"my": "payload"}) }}"
```
returns _eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJteSI6InBheWxvYWQifQ.DgiGYk5rNAcEXTRX3cYOMPIIhqTvvlFlVxaEHJzKasw_

### not_

Returns a boolean with the opposite value of the parameter

params:
* value: the boolean value to apply the not operand
i.e.
```
"{{ $not_(true) }}"
```
returns _false_

### pretty

Formats a JSON content in a pretty format.

params:
* content: the JSON to be formatted
i.e.
```
"{{ $pretty('{"a":"value"}) }}"
```
returns 
```
{
    "a":"b"
}
```

### port

Returns the port of an URL

params:
* url: from where extracting the host
i.e.
```
"{{ $port('https://my.url.com/path') }}"
```
returns _443_

### query_param

Returns the value of a query param in a URL

params:
* name: name of the param to search for
* url: url where searching for

i.e:
```
"{{ $query_param('name', 'http://my.url.com?name=value&name2=value3') }}"
```
returns _value_

### query_string

Returns the query string of an URL

params:
* url: from where extracting the query string
i.e.
```
"{{ $query_string('https://my.url.com/path?a=b&c?d') }}"
```
returns _a=b&c=d_

### saml_id

Returns the ID of a SAMLRequest payload

params:
* SAMLRequest: the SAML request zlib and base64 encoded
i.e.
```
"{{ $saml_id('jVJdb9owFP0rkd8TJ2F8xEpSsXXVKhUNAWXSXibjXIq32Ka+Tmj//RwStPKC9np9z4fPufndm6qDFixKowuSRDEJQAtTSf1SkOfNQzgjd2WOXNXpkc0bd9AreG0AXeCBGln/UpDGamY4SmSaK0DmBFvPF08sjWJ2tMYZYWoSzBHBOi/1xWhsFNg12FYKeF49FeTg3BEZpdyrRCDCo4Vox+UfXh9r7vbGqkgYRTvFcL3+ToJ7b0Nq7s7WL+hT2+GUaSU6bjuE56ILuRgmv35sKQkejBVw/k5B9rxGIMHjfUE4TCbZy++d2EFWTaej6X4EM7nPKphUU7+CS44oW/gHQmzgUXti7QqSxkkWxmmYJJskZaOMJUk0G49/kmA5RPBZ6j7aW3nt+iVk3zabZbiCSloQjgTbS01+iQylsLMD+7GN2+T8UgEp/zPwnH5UKnNVsa9vDnTnBQdhVd1WVeB4xR33ti28srmwW177MxrgftY32BV4OkXtFEE0Vrr3vnJxAMWpj3dM40/U9hdIylFOr9jKnF6ZKwfr15db/gU=')}} "
```
returns _ae669gjbcbe9d7737f3e8if9de6d7_

### scheme

Returns the URL schema

params:
* url: from where extract the schema
i.e.
```
"{{ $schema('https://my.url.com') }}"
```
returns _https_

### scrape

Takes an HTML form parameter or action from an HTML content

params:
* field: the field to fetch its value. It can be
    - 'input.<name>': returns the value of the form input field with that name
    - 'select.<name>': returns the first option of a form select field with that name
    - 'form.action': returns the action of the first found form
* content: html content where searching

i.e:
```
"{{ $scrape('input.myinputfield', previous-stage.response.body) }}"
```
 searches for a form input named 'myinputfield' into the body returned in previous-stage

### strip_fragment

Returns the URL without fragments. If the URL does not have fragment, returns the original URL

params:
* url: from where strip the fragment
i.e.
```
"{{ $strip_fragment('http://my.url.com#myfragment?lalala=lololo') }}"
```
returns _http://my.url.com_

### url_encode

Encodes values as an URL.

params:
* content: the string to be encoded
i.e.
```
"{{ $url_encode('bisca barça')}"
```
returns _bisca%20bar%C3%A7a_

### url_decode

Decodes an URL encoded value

params:
* content: the string to be decoded
i.e.
```
"{{ $url_decode('bisca%20bar%C3%A7a') }}"
```
returns _bisca barça_

### uuid

Returns a new universal unique identifier

i.e.
```
"{{ $uuid() }}"
```
could return, for instance, _550e8400-e29b-41d4-a716-446655440000_

### slack

Fetches an OTP code from a certain slack channel

mandatory params:
* access_token: the oauth token for the script to access the channel. You can
    generate a token [here](https://api.slack.com/apps) for an application that can 
    access the desired channel. Or you can launch apichk with the slack spec
    included in this distribution for the same purspose
* channel: the channel identifier where the OTP is expected to be added as message.
    Note that this is not the name but the identifier given by slack for such channel.
* msisdn: the phone number where the code is sent

optional parameters:
* fallback_msg: the message to propt in case there is not configured token,
    the token is expired or invalid.

i.e.
```
$slack('xoxp-120723272067-519143629842-548815047891-3843b79c030a3e7ea0acbad7b46', 'CE806NM26', '689295859', 'Please enter OTP: ')
```
returns the code, like for instance, _9999_

> Note: See [Providers](# Providers) for more info

## Nested Functions

Functions can be nested:

```
"{{ $concat($schema(previous-stage.response.body.server-url), '://', $hostname(another-stage.response.body.other-url)) }}
```
concats the schema of _previous-stage.response.body.server-url_ with _://_ literal with _another-stage.response.body.other-url_ hostname

## Embedded Functions

Sometimes is difficult or even impossible to nest functions, placeholders
or literals to get the desired result. In these cases we can define
funcions directly into the yaml structure so that the name of the 
function is a node preceded with the *$* symbol and taking a list
of subnodes as parameters, like this:
```
$the_function:
    - param1
    - param2=value2
    - param3
```
The children list of parameters are in the order to be taken by the function.
Every parameter is taken as a string parameter and can hold a placeholder
that will be resolved before the function execution. An example of this:
```
 $jwt_encode:
    - "{{ get-jwt-private-key.response.body }}"
    - iss: MiMovistar_WebViews
    args:
        documentoId: "{{ config.username }}"
        clave: "{{ config.password }}"
        cb: "{{ config.saml-sso-url }}"
        idSAML: "{{ $saml_id($url_decode($query_param('SAMLRequest', saml-login.response.headers.location))) }}"
    - iat
    - exp=600
```
all the parameters being placeholders are resolved before _jwt_encode_ function
is executed. 
*The result of embedded function execution replaces the node where the 
function is defined*. So, in this case, once the JWT encoding happens,
the node having $jwt_encode is replaced by the result

# Raw HTTP Messages

Executint the tool with *-v* flag dumps out to console the last HTTP
raw request and response in case that execution ends with an error.
Regardles of the final result, if you want to obtain all the HTTP messages
for all the executed stages, they are written to the file */tmp/apichk_msgs.log*.
If any response includes HTML content, it is not shown by default
in the raw messages log. Instead, you should see the label:

```
--- <the HTML content goes here> ---
```

If you really want to include
into the traces the whole HTML content, you have to execute _apichk_
with the *-vv* modifier

There is another modifiler *-l* (aka *--logs*) which allows you to indicate
a different folder where storing log files other than the default _/tmp_ one

# Quiet mode

Maybe you desire not to show any progress on the intermediate stages but
dump out the final results because you are only interested in the 
generated result or in publishing some of them. In such case you can
add the *-q* or  *--quiet* modifier to execute the tool in quiet mode.

> Note: 
>   positive verbose modifiers have precedence over quiet mode
>   so that any _-v_ _-vv_ or _--verbose_ modifiers void any quiet ones
