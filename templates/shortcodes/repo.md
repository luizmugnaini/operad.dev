{%- set label = text | default(value=name) -%}
[{{ label }}]({{ config.extra.code_url }}/{{ name }})
