# Home Assistant Automation ID Generator

Small native Home Assistant script to generate the next available automation ID based on a prefix.

No custom integration required.

## Why?

Home Assistant does not provide a native way to generate consistent automation IDs.  
This script scans existing automations and returns the next available ID.

## Helpers

Create these helpers:

input_text.prefix_id  
input_text.next_id_result

## Script

Add the script from `script.yaml`.

## Usage

1. Enter a numeric prefix in `input_text.prefix_id`
2. Run the script
3. The next available automation ID will appear in `input_text.next_id_result`

Example:

Prefix: 1012
Generated ID: 10120000000015


## Notes

- IDs must be numeric
- The script ignores malformed automation IDs
- Works with YAML and UI automations

## License

MIT
