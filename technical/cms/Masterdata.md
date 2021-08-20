# Masterdata

Masterdata can often be likened to an enumeration. You have for example contract types or product categories or log levels or ...
So instead of having a free text field in a particular table, you instead have a fixed list of allowed values and your table contains a foreign key to the selected value.

This has some advantages:

- common naming: if you allow free text, even with a distinct enumerated suggestion, the same value might appear multiple times with different methods of writing it (capitalization, numbers, whitespace, language...)
- translations: masterdata has out of the box translation support for easy visualization
- resolving: there is automatic resolving for masterdata in page builder, adding a custom resolver is not all that difficult though so it's a tiny quality of life improvement
- renaming: renaming masterdata can be done in a single central record rather than all the places it is used.
- logic: building logic on masterdata (e.g. branching business logic) is more stable than if you are simply using free text

Masterdata has two distinct values:

- name: static, shouldn't change if at all possible, you can build business logic on top of this. We usually use camelCase notation
- label: a human readable version of the masterdata. This is based on the name initially but can be altered with translations.

## Extensions

Sometimes you want to have the advantages of masterdata but still add some custom values, you can extend the masterdata table if necessary to add more complex masterdata.
However, the tradeoff has to be clear, if you are not particularly benefitting from any available masterdata system, you can consider creating a completely separate table that is not extending masterdata.

## Preloading
