---
name: espos-registry-components
description: Why an IDF component's optional dependencies and feature checks silently stop working when the component is installed from the ESP Component Registry instead of a checkout — namespaced target names, and the idf_component_optional_requires defect that follows from them.
---

# Component checks that survive a registry install

Verified against ESP-IDF **v6.0.3**, September 2026.

## The failure

A component behaves one way from a git checkout and differently when a consumer
installs it from the ESP Component Registry — with **no error and no warning**. Code
takes its "that component is not in this build" branch, features compile out, and the
build goes green.

Cause: `BUILD_COMPONENTS` holds CMake **target** names, and the component manager
namespaces those on install. A component published as `myns/mycomp` builds as target
`myns__mycomp`, never the bare `mycomp`. So:

```cmake
idf_build_get_property(comps BUILD_COMPONENTS)
if(mycomp IN_LIST comps)          # true in-tree, silently FALSE from the registry
    target_compile_definitions(${COMPONENT_LIB} PRIVATE HAVE_MYCOMP=1)
endif()
```

One real instance: a firmware's one-call startup function decided at configure time
which optional subsystems existed. Installed from the registry it brought up no WiFi,
no data client and no OTA on a build that had linked all three. Nothing failed; the
features simply were not compiled in. The proof is blunt — grep the build's
`compile_commands.json` for your define and find it absent.

## `idf_component_optional_requires()` has the same defect

This one is in IDF itself, not in your component
(`tools/cmake/component.cmake`, v6.0.3):

```cmake
function(idf_component_optional_requires req_type)
    idf_build_get_property(build_components BUILD_COMPONENTS)
    foreach(req ${optional_reqs})
        if(req IN_LIST build_components)      # <-- target names again
```

So an optional dependency named by its bare name is **silently skipped** for a
namespaced install. The consequence is indirect and easy to misattribute: the optional
requirement does not link, so that component's headers are not on the include path, so
a `__has_include("othercomp.h")` guard answers no, and a REST handler (or whatever it
gated) compiles to a stub. The symptom is a 404 at runtime; the cause is three steps
away in CMake.

## The fix

Match on the entries of `BUILD_COMPONENTS` themselves — exactly, or on a `__<name>`
suffix. Suffix-match rather than stripping a literal namespace, so a fork or mirror
published under a different namespace keeps working:

```cmake
# In a file each component ships and includes itself (see "Where to put it").
if(COMMAND espos_has_component)
    return()
endif()

function(espos_has_component var name)
    idf_build_get_property(_comps BUILD_COMPONENTS)
    foreach(_t ${_comps})
        if(_t STREQUAL name OR _t MATCHES "__${name}$")
            set(${var} TRUE PARENT_SCOPE)
            if(ARGC GREATER 2)
                set(${ARGV2} ${_t} PARENT_SCOPE)   # the name THIS build uses
            endif()
            return()
        endif()
    endforeach()
    set(${var} FALSE PARENT_SCOPE)
    if(ARGC GREATER 2)
        set(${ARGV2} "" PARENT_SCOPE)
    endif()
endfunction()

function(espos_optional_requires req_type)
    foreach(_req ${ARGN})
        espos_has_component(_present ${_req} _real)
        if(_present)
            idf_component_get_property(_lib ${_real} COMPONENT_LIB)
            target_link_libraries(${COMPONENT_LIB} ${req_type} ${_lib})
        endif()
    endforeach()
endfunction()
```

**Pass the resolved name, not the bare one**, to `idf_component_get_property()`. It
raises a `FATAL_ERROR` on a name it cannot resolve, and its own bare-name fallback
works only once that component has been registered — which is not guaranteed at the
point these run. Verified the hard way: passing the bare name failed with
`Failed to resolve component 'espos_sk'`.

Names that already carry a namespace (`espressif__cjson`) pass through unchanged,
matched literally by the same helper.

## Where to put it

Each component that calls the helpers needs **its own copy**, because a component
installed from the registry cannot reach a sibling's directory. Ship
`cmake/<yourns>_components.cmake` inside every such component and include it from that
component's own `CMakeLists.txt`, before the first call:

```cmake
include("${CMAKE_CURRENT_LIST_DIR}/cmake/myns_components.cmake")
```

Two mistakes worth avoiding, both of which produce a hard build failure rather than a
silent one:

* **Do not define the helpers only in one component's `project_include.cmake`.** That
  file runs only when *that* component is in the build, so any other component calling
  the helper dies with `Unknown CMake command`. Reproduced.
* **Include before the first use.** `project_include.cmake` running earlier can mask a
  missing include in-tree and let it fail elsewhere.

The `if(COMMAND ...) return()` guard at the top is what makes several components
including their own copies harmless.

## Your CI probably cannot catch this

This is the part worth internalising. If your "install from the registry" example uses
`override_path` to build against the local tree — which is the normal way to keep such
an example honest in-repo — then it builds under the **bare** names too. The one
configuration that would notice the bug is the one CI does not have.

So add a cheap static check instead, and run it in CI:

* fail on `\b<yourprefix>_[a-z0-9_]+\s+IN_LIST\b` in any `CMakeLists.txt`,
  `project_include.cmake` or component `cmake/*.cmake`
* match an **optional quoted** name: `if("mycomp" IN_LIST comps)` is the identical
  defect, since CMake treats the quoted form as the same literal element
* use `re.S` / multiline matching: CMake lets a condition wrap, and
  `if(mycomp\n   IN_LIST comps)` is the same bug split over two lines
* if components ship duplicate copies of the helper file, assert they are
  **byte-identical** — duplication that drifts reintroduces exactly what the check
  exists to prevent

To reproduce a registry layout locally without publishing anything: copy the
components into `components/<namespace>__<name>/` and **strip `override_path` from
every manifest**. Then check your defines actually appear in
`build/compile_commands.json`.

## Related

`espos-firmware-releases` covers the other half of publishing — getting release
images to a place a browser can fetch them.
