This guide is intended to help fill gaps in the documentation of functions for an Elixir module implementing GenServer-based tasks.

## Building Call Flows

Take a look at the module and build all possible chains of calls. One chain of calls consists of:
- `init()` or a public interface of the module - the function which is intended to be called outside of the module. This function usually uses GenServer's call or cast facility to invoke the module functionality.
- `handle_call`, `handle_cast` or `handle_info` function which is initiated by the public interface. The interface and the `handle_cast`/`handle_call`/`handle_info` usually can be matched by the atom which is the first element in the tuple of the `handle_cast`/`handle_call`/`handle_info`.
- The rest of calls of the module functions which are initiated by `handle_cast`/`handle_call`/`handle_info`. Only functions belonging to the inspected module must be listed here in the order they construct the flow of calls.

### Call Flow Examples

1. Example for the most common case:

**Interface:** `inc_error_count/3`
**Handler:** `handle_cast({:inc_error_count, url, json_rpc_named_arguments, url_type}, state)`
**Further call flows:**
```
..do_increase_error_counts
....log_url_unavailable
......available_urls
........do_filter_unavailable_urls
......fallback_url_set?
```

2. Example for a call flow performed during initialization:

**Intialization:** `init/1`
**Further call flows:**
```
..func1
....func2
..func3
```

3. If a handler `handle_info` is invoked due to a timer expiration, specify where the timer was set:

**Timer setup:**
```
..init
....schedule_next_cleaning
```
**Handler:** `handle_info(:clear_old_records, state)`
**Further call flows:**
```
..do_clear_old_records
..schedule_next_cleaning
```

## Build Functions List

1. Take a look at all call flows you built. Build a list of all public and private functions (except `handle_call`, `handle_cast`, and `handle_info`) that participate in these call flows. The functions that terminate the branches of the call flows should be listed first.

Example, consider these two call flows:

```
..do_increase_error_counts
....now
....log_url_unavailable
......available_urls
........do_filter_unavailable_urls
......fallback_url_set?
```

and

```
..log_url_available
....available_urls
......do_filter_unavailable_urls
....fallback_url_set?
```

Based on the call flows above, here is the list of functions that need to be analyzed:

- `do_filter_unavailable_urls/2`
- `fallback_url_set?/2`
- `available_urls/3`
- `log_url_unavailable/4`
- `log_url_available/4`
- `now/0`
- `do_increase_error_counts/4`

2. List all GenServer handlers (`handle_call`, `handle_cast` and `handle_info` but skip `init` and `start_link`) with their full clauses but without implementation. Include everything between `def` and `do`.

## Specifications Improvement

Check specifications for every private and public function you listed, starting from the first function in the prepared list. For each function:
   - Verify if a specification exists
   - Evaluate if the existing specification could be improved
   - Pay special attention to private functions, which often lack specifications
   - Suggest changes for any missing or improvable specifications

## Functions Documentation Improvement

1. Review documentation comments for each private function in your prepared list, starting from the first one. For each function:
   - Check if documentation comment exists
   - Evaluate if existing documentation could be improved
   - If the function calls methods from modules not in the current context, request those modules before writing documentation

2. Review documentation comments for all GenServer handlers in your list. For each handler:
   - Check if documentation comment exists
   - Evaluate if existing documentation could be improved
   - If the handler uses modules not in the current context, request those modules before writing documentation

3. Review documentation comments for each public function in your list. For each function:
   - Check if documentation comment exists
   - Evaluate if existing documentation could be improved
   - If the function uses modules not in the current context, request those modules before writing documentation

## Module Documentation

Review and update the module documentation. Based on the module's public functions:
1. Identify the main usage scenarios and patterns
2. Document these scenarios with clear examples
3. For GenServer modules:
   - Document the state structure and how it changes
   - Show how state changes relate to key interaction scenarios
4. Ensure documentation reflects the module's actual behavior and capabilities
