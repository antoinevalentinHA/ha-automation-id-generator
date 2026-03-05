# Home Assistant Automation ID Generator

A small **native Home Assistant script** that generates the **next available automation ID** from a numeric prefix.

No custom integrations required.

This tool scans existing automations in Home Assistant, extracts their IDs, and calculates the next available one for the selected prefix.

---

# Why?

Home Assistant does not provide a built-in way to generate consistent automation IDs.

When managing many automations, IDs can quickly become:

* inconsistent
* duplicated
* difficult to maintain

This script solves the problem by automatically generating the next available ID based on a prefix.

Example structure:

```
1001xxxxxxx  → Lighting
1002xxxxxxx  → Security
1012xxxxxxx  → System
```

The script scans all automations and determines the **next valid suffix**.

---

# Requirements

This solution uses only native Home Assistant components:

* Script
* Helpers (`input_text`, `input_select`)
* Optional Lovelace dashboard

No custom integrations are required.

Compatible with **Home Assistant 2024.8+**.

---

# Installation

## 1. Create Helpers

Add the following helpers to your configuration:

```yaml
input_text:

  prefix_id:
    name: Automation ID prefix
    pattern: '^[0-9]+$'
    mode: text
    max: 10

  next_id_result:
    name: Next automation ID
    mode: text
    max: 20


input_select:

  prefix_id_select:
    name: Automation prefix
    options:
      - "1001 - example_domain_1"
      - "1002 - example_domain_2"
      - "1012 - system"
    icon: mdi:numeric
```

The `input_select` provides a convenient way to choose a domain prefix.

---

## 2. Add the Script

Add the script from `scripts.yaml`:

```yaml
generate_next_id_from_prefix:
  alias: "Generate next automation ID from prefix"
  mode: single

  sequence:

    - variables:
        prefix: "{{ (states('input_text.prefix_id') | default('') | trim) | string }}"

    - choose:
        - conditions:
            - condition: template
              value_template: "{{ prefix in ['unknown','unavailable','none','', None] }}"
          sequence:
            - stop: "Prefix empty"

    - choose:
        - conditions:
            - condition: template
              value_template: "{{ not (prefix | regex_match('^[0-9]+$')) }}"
          sequence:
            - stop: "Prefix must be numeric"

    - variables:
        suffixes: >-
          {% set ns = namespace(ids=[]) %}
          {% set p = (prefix | string) %}
          {% for s in states.automation %}
            {% set aid = (state_attr(s.entity_id, 'id') | default('') | string) %}
            {% if aid.startswith(p) %}
              {% set tail = aid[p | length:] %}
              {% if tail | regex_match('^[0-9]+$') %}
                {% set ns.ids = ns.ids + [tail | int] %}
              {% endif %}
            {% endif %}
          {% endfor %}
          {{ ns.ids }}

    - variables:
        next_suffix: "{{ (suffixes | max + 1) if (suffixes | length > 0) else 1 }}"
        next_id: "{{ prefix }}{{ '%010d' | format(next_suffix) }}"

    - service: input_text.set_value
      target:
        entity_id: input_text.next_id_result
      data:
        value: "{{ next_id }}"
```

---

## 3. Sync the Prefix Selector (Optional but recommended)

This automation copies the prefix from the `input_select` helper into the working helper `input_text.prefix_id`.

```yaml
- alias: "Sync prefix select to input_text.prefix_id"

  trigger:
    - platform: state
      entity_id: input_select.prefix_id_select

  action:
    - service: input_text.set_value
      target:
        entity_id: input_text.prefix_id
      data:
        value: "{{ states('input_select.prefix_id_select').split(' - ')[0] | trim }}"
```

---

# Usage

1. Select a prefix in `input_select.prefix_id_select`
2. Run the script `generate_next_id_from_prefix`
3. The generated ID will appear in:

```
input_text.next_id_result
```

Example:

```
Prefix:      1012
Generated:   10120000000015
```

---

# Optional Dashboard Example

You can create a simple Lovelace dashboard to make the tool easier to use.

```yaml
title: Automation ID Generator
icon: mdi:numeric

views:
  - title: Automation IDs
    path: automation-ids
    icon: mdi:format-list-numbered

    cards:
      - type: vertical-stack
        cards:

          - type: markdown
            content: |
              ## Automation ID Generator
              Select a prefix and generate the next available automation ID.

          - type: tile
            entity: input_select.prefix_id_select
            name: Automation domain
            features_position: bottom
            features:
              - type: select-options

          - type: button
            name: Generate next ID
            icon: mdi:calculator
            tap_action:
              action: call-service
              service: script.generate_next_id_from_prefix

          - type: tile
            entity: input_text.next_id_result
            name: Next available ID
```

---

# Notes

* Prefix must be numeric
* The script ignores malformed automation IDs
* Works with both **YAML automations** and **UI-created automations**
* The suffix is padded to **10 digits by default**
* Prefix format can be adapted to your automation structure

---

# License

MIT
