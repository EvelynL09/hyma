# HYMA Web Server

This is a UCLA CS130 course project developed by Team HYMA.
You can access our cloud-deployed server via [http://www.hyma.cs130.org.](http://www.hyma.cs130.org.)

## Table of Contents

- [Getting Started](#getting-started)
  * [Environment Setup](#environment-setup)
  * [Build](#build)
  * [Run Tests](#run-tests)
  * [Start the Server](#start-the-server)
  * [Generate Code Coverage Report](#generate-code-coverage-report)
  * [Deployment](#deployment)
- [Contributing](#contributing)
  * [Project Files Structure](#project-files-structure)
  * [Add New Request Handler](#add-new-request-handler)
    + [Directory Layout](#directory-layout)
    + [Request Handler Interface](#header-file-of-request-handler)
    + [Example of Existing Handler](#example-of-existing-handler)
    + [Step-By-Step Guide](#step_by_step-guide)
- [Built With](#built-with)
- [Team Member](#team-members)
- [References](#references)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'> \* Table of contents generated with markdown-toc</a></i></small>

---
**Quick Links**:

- [How is the source code laid out?](#project-files-structure)
- How to [build](#build), [test](#run-tests), [and run](#start-the-server) the code
- [How to add a new request handler?](#add-new-request-handler)
    - [An example of an existing handler](#example-of-existing-handler)
    - Header Files
        - [Header File of Request Handler](#header-file-of-request-handler)
        - [Header File of Echo Request Handler](#header-file-of-echo-request-handler)
---
## Getting Started

### Environment Setup

[Assignment 1 - Envrionment Setup](https://www.cs130.org/assignments/1/#environment-setup)

### Build
In the project root directory `/hyma`, run the following command:
```bash
$ mkdir build && cd build
$ cmake .. && make
```
### Run Tests
To run all the tests, including unit tests and the integration test: `ctest` or `make test`.

To run an individual unit test (e.g. config_parser_test), run
` cd ../tests && ../build/bin/config_parser_test && cd ..` in build directory

To run the integration test, run
` ../tests/integration_test.sh` in `/build` directory
### Start the Server
In build directory,
```
$ ./bin/webserver ../config/example_config    
```
Now you can access our webserver through localhost:8080.
### Generate Code Coverage Report
In project root directory, run the following command:
```bash
$ mkdir build_coverage
$ cmake -DCMAKE_BUILD_TYPE=Coverage ..  && make coverage
```
### Deployment
In project root directory, build the docker image:
```
$ docker build -f docker/base.Dockerfile -t hyma:base .
$ docker build -f docker/Dockerfile -t hyma:latest .
```
Start the server:
```
$ docker run --rm -p 80:80 --name my_server hyma:latest
```
Now, you can access the web server through localhost:80.
For more details about deployment, check out
[Adrian's Deployment Note](Deployment_Note.md)

---
## Contributing
### Project Files Structure
##### Overview
(Scroll down to the `src/` subsection in the Details section below if you only want to see descriptions of the .cc source files.)
```bash
hyma/
├── CMakeLists.txt
├── .md...     #Any Markdown Notes
├── config/    #Web Server Config Files
├── docker/    #Docker Related Files
├── include/   #Header Files ("xx.h")
├── log/       #Server's Log Files
├── public/    #Static Files that our server serves
├── src/       #Source Files ("xx.cc")
└── tests/     #Unit Tests and Integration Tests

```
##### Details
- `CMakeLists.txt`
    - This file is the input to the CMake build system for building this project
    - It contains a set of directives and instructions describing the project's source files and targets (executable, library, or both).
- `config/`
    - `example_config`
        - The config file that starts the server at port 8080 for local development.
            ```nginx
            port 8080;
            ```
        - It contains location statments that describes URLs and request handlers mapping. The grammar is a bit similar to Nginx.
        - A location statement should looks like the following (*the "root" token is only required for StaticHandler*):
            ```nginx
            location "A URL path" TypeHandler {
                root "file directory path";
                #Add your comment with a '#'
            }
            ```
        - Here's an example,
            ```nginx
            location "/static/" StaticHandler {
                root "../public/html";
                # hyma/public/html/...
                #Add comment with '#'
            }
            ```
            - For a configurable statement like this, once you start the server, you can go to [http://localhost:80/static](http://localhost:80/static) or [http://localhost:8080/static](http://localhost:80/static) on your web browser to access `/hyma/public/index.html`
    - `gcp_config`
        - The config file that used for docker/cloud build. It starts the server at port 80.
        - Everything except the port number should be same with `example_config`.
- `docker/`
    - base.Dockerfile
        - It contains commands used to assemble Base container image
        - Update the base image and install build environment
    - cloudbuild.yaml
        - Cloud Build configuration file
        - Details are available at [Assignment 3: Set Up a Continuous Build](https://www.cs130.org/assignments/3/#set-up-a-continuous-build)
    - coverage.Dockerfile
        - This is for test coverage build.
    - Dockerfile
        - This is for building `hyma:latest` image
    - For more details, check out [Deployment Section](#deployment) or [Assignment 2: create a docker container](https://www.cs130.org/assignments/2/#create-a-docker-container)

- `src/`
    - `server_main.cc`
        - class containing `main()`
        - It starts the server with the given config file and set up the logging (for both console and files)
    - `config_parser.cc`
        - config_parser class parses config file
        - It extracts port number information
        - All parsed statements are stored as `NginxConfigStatement`
    - `server.cc`
        - server class waits for incoming connections from clients and asynchronously creates a socket and session upon receiving a connection
    - `session.cc`
        - session class manages each single session
        - It is in charge of asynchronous read and write.
        - It also fetches client's IP address for logging
    - `http/`
        - `handler_dispatcher.cc`
            - handler_dispatcher class initializes all of the request handlers needed for the server after parsing the NginxConfig object passed to its constructor.
            - It maintains an `std::unordered_map<std::string, request_handler_ptr>`and fetches a handler from this map when the session objects ask for one.
        - `mime_types.cc`
            - It describles the mapping of static files type and `Content-Type` field for an HTTP response.
            - For example, a zip file should has the `Content-Type` field as `application/zip`
            - We currently support `.gif`, `.htm`, `.html`, `.jpg/.jpeg`, `.png`, `.pdf`, and `.zip` files.
        - `request_handler_404.cc`
            - 404 handler always returns a HTTP 404 (not found) response
        - `request_handler_health.cc`
            - Health handler always returns a HTTP 200 (OK) response with plain text payload "OK"
            - For server's health monitoring
        - `request_handler_echo.cc`
            - Echo handler would echo whatever request it receives in the response body
        - `request_handler_file.cc`
            - Static file handler fetches a file from the server's filesystem and returns it to the client
        - `request_handler_status.cc`
            - Status handler returns a status message (e.g. how many total requests has the server received, and what request handlers exist at which url prefixes)
        - `request_parser.cc`
            - Request parser parses an HTTP request and determines whether it is valid, invalid, or indeterminate (i.e. need to read more to determine validity)
        - `response_utl.cc`
            - A utility class that contains helper methods needed for response creation

- `include/`
    - `http/`
        - `request_handler.h`
            - This header file defines the interface for all request handlers
            - All request handlers are inherited from this class
            - It describes a pure virtual function that all its child classes should have
            ```cpp
            virtual response handle_request(const request& req) = 0;
            ```
    - rest files are of simlilar struture as `src/` and all corresponding `.h` files are stored here
- `tests/`
    - similar structure as `src/`
    - contains all corresponding `class_test.cc` files with all the unit tests


### Add New Request Handler
#### Directory Layout
```bash
hyma
├── config    #Add "location statment" for new handler
│   ├── example_config #to both example_config (for local build)
│   └── gcp_config     #        and gcp_config (for cloud build)
├── include   
│   └── http  #Add new handler's header file here
├── src       
│   └── http  #Add new handler's source code here
└── tests     
    └── http  #Add the unit test for new handler here
```
#### Header File of Request Handler
```cpp
#ifndef HTTP_REQUEST_HANDLER_HPP
#define HTTP_REQUEST_HANDLER_HPP

#include <boost/shared_ptr.hpp>
#include <string>
#include "http/request.h"
#include "http/response.h"

namespace http {
namespace server {

// a common base request handler interface for request handlers implementations:
// design of this abstract class in C++ is followed by the guideline:
// https://www.tutorialspoint.com/cplusplus/cpp_interfaces.htm
class request_handler
{
public:
  // All subclasses must implement a static construction method.
  // Pass in the config location and scoped block of arguments.
  // static request_handler* init(const string& location_path, const NginxConfig& config);

  // handle_request() is a so-called `pure virtual function`:
  // the inheriting class is required to implement
  // the pure virtual function. Also, we cannot implement this
  // function in the base class.
  virtual response handle_request(const request& req) = 0;
};

typedef std::shared_ptr<request_handler> request_handler_ptr;

} // namespace server
} // namespace http

#endif // HTTP_REQUEST_HANDLER_HPP
```
#### Example of Existing Handler
Below are the contents of request_handler_echo.h. You can see that we have, from top to bottom...
* Include guards (`#ifndef` and `#define`, followed by `#endif` at the very end) so that we prevent circular includes
* Include statements of required header files
* Declaration of http and server namespaces (this is a convention followed by Boost examples, putting webserver-related classes inside the same namespace)
* Class declaration, which includes inheritance from the request_handler interface, a constructor that sets the location_path_ member variable equal to its single argument (needed particularly for request handlers that fetch an object on the server), a destructor (currently empty), a static `init` function for construction (required by Common API) and a `handle_request` function (also required by Common API).
##### Header File of Echo Request Handler
```c++
/*
    File Path: /hyma/include/http/request_handler_echo.h
*/
#ifndef HTTP_REQUEST_HANDLER_ECHO_HPP
#define HTTP_REQUEST_HANDLER_ECHO_HPP

#include <string>
#include "config_parser.h"
#include "http/request.h"
#include "http/response.h"
#include "http/request_handler.h"

namespace http {
namespace server {

// The handler for incoming requests of echoing purposes
class request_handler_echo : public request_handler
{
public:
  request_handler_echo(const std::string& location_path) // inherits the constructor from its base class
    :request_handler::request_handler()
  {
    location_path_ = location_path;
  };

  ~request_handler_echo(){};

  static request_handler* init(const std::string& location_path, const NginxConfig& config);
  response handle_request(const request& req);
private:
  // The location path/prefix used to fetch the handler (e.g. "/echo")
  std::string location_path_;
};

} // namespace server
} // namespace http

#endif // HTTP_REQUEST_HANDLER_ECHO_HPP
```

Below are the contents of request_handler_echo.cc. You can see that we have, from top to bottom...
* Include statement for the handler's header file
* http and server namespace declarations, as in the header file
* Implementation of the static `init` function for handler construction, which takes a location path and a locally scoped NginxConfig object. Here, the NginxConfig object is not used because the echo handler does not need to know a base directory since it does not fetch any files from the server's filesystem.
* Implementation of the handle_request() virtual function that is inherited from the request_handler interface. This takes a request object and returns a response object.
##### Source File of Echo Request Handler

```c++
/*
    File Path: /hyma/src/http/request_handler_echo.cc
*/

#include "http/request_handler_echo.h"

namespace http {
namespace server {

request_handler* request_handler_echo::init(const std::string& location_path, const NginxConfig& config) {
  return new request_handler_echo(location_path);
}

response request_handler_echo::handle_request(const request& req)
{
  // Return a response with the request in the body.
  response rep;
  rep.status = response::ok;
  rep.content = req.body;  // The body of request is replaced by the whole incoming request in handle_read method from session.cc
  rep.headers.insert(std::pair<std::string, std::string>("Content-Type", "text/plain"));
  rep.headers.insert(std::pair<std::string, std::string>("Content-Length", std::to_string(rep.content.size())));
  return rep;
}

} // namespace server
} // namespace http
```

#### Step-By-Step Guide
We will refer to this new request handler as "request_handler_NEW" in the following steps.

1. Create a new header file, request_handler_NEW.h, inside the hyma/include/http/ directory.

2. Copy and paste the contents of request_handler_echo.h (found inside hyma/include/http/) inside request_handler_NEW.h. Then, rename all of the “request_handler_echo” keywords to "request_handler_NEW", including the constructor/destructor AND the include guards (the lines with `#ifndef` and `#define` and `endif`).

3. Create a new source code file, request_handler_NEW.cc, inside the hyma/src/http/ directory.

4. Copy and paste the contents of request_handler_echo.cpp (found inside hyma/src/http/) inside request_handler_NEW.cc. Then, rename all of the “request_handler_echo” keywords to "request_handler_NEW". Change the implementations of the init() and handle_request() functions to create the appropriate functionality for this new request handler. Create private helper functions/variables inside this new class if needed.

5. Add a new location statement to both of the 2 config files, hyma/config/example_config and hyma/config/gcp_config, as follows
```
location "/NEW" NEWHandler { # replace the "NEW" with an appropriate name of your choice
}
```

6. In hyma/src/http/handler_dispatcher.cc, add an include statement: `#include "http/request_handler_NEW.h"`. Then go to the definition for the create_handler() function inside this file, and add a new "else if" block before the "else" statement in the same manner as the previous "else if" blocks, as follows:
```c++
else if (handler_type == "NEWHandler") { // Replace "NEWHandler" w/ actual keyword used in config file
  std::shared_ptr<request_handler> ret(request_handler_NEW::init(location_path, config));
  return ret;
}
```

7. In hyma/include/session.h, add an `include` statement to include this new class: `#include "http/request_handler_status.h"`.

8. Create a new unit test file, request_handler_NEW_test.cc, inside the hyma/tests/http/ directory.

9. Copy and paste the contents of request_handler_echo_test.cpp (found inside hyma/tests/http/) inside request_handler_NEW.cc. Then, rename all of the “request_handler_echo” keywords to "request_handler_NEW". Change the test fixture to your own liking, and create your own unit tests whenever appropriate.

10. In hyma/CMakeLists.txt, add the following lines (try to organize them in the right sections—you will see a bunch of `add_library` statements in one place, a bunch of `add_executable` in another place, and so on):
```
add_library(request_handler_NEW src/http/request_handler_NEW.cc)
...
add_executable(request_handler_NEW_test tests/http/request_handler_NEW_test.cc)
target_link_libraries(request_handler_NEW_test request_handler_NEW gtest_main Boost::system Boost::filesystem)
...
gtest_discover_tests(request_handler_NEW_test WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/tests)
```

11. In hyma/CMakeLists.txt, search for a line that starts with `set(HANDLER_LIBS `. These are the handler libraries; add `request_handler_NEW` into this line. Then, search for a line that starts with `generate_coverage_report(TARGETS` and you will see that it has `TARGETS` and `TESTS`; in the `TARGETS` section, add `request_handler_NEW`, and in the `TESTS` section, add `request_handler_NEW_test`.

12. Create a new integration test for this new handler in hyma/tests/integration_test.sh. You can copy and paste one test case (test cases are divided by the `echo "-----------------------"` statements) to the end of the file and modify it as you please.

---
## Built With
- Boost C++
- CMake
- Google Test
- Docker
- Google Cloud Platform
- ...
---
## Team Members
- Haiwei Lu
- Yifu Yuan
- Moo Jin Kim
- Adrian Hsu
---
## References
