# per-form

A form-state/validation/list-editing library, forked from [octant/per-form](https://github.com/octant/per-form)
and substantially extended (function-message validations, repeating-row
`list` fields with per-row commit gating, subform types). Consumed as source
(no build step) by apps that clone this repo directly into `src/lib/per-form`.

per-form owns form **state** (values, dirty/touched, validation, submit-time
conversion) and renders nothing itself: a *template* component renders each
field. The Fluent UI template is
[typical-admin-fabric](../typical-admin-fabric)'s `lib/templates/Fabric.tsx`,
wrapped as its `Form` component; [typical-admin](../typical-admin) has a bare
HTML one (`lib/templates/Raw.tsx`).

MIT licensed, per the original upstream project; see `LICENSE`.

## Using with an AI coding agent

Before writing any code with these libraries, read the skills in
`typical-admin-fabric/.claude/skills/`, in particular `form-schema` (every
field type and property, what is dead, and a validator script) and
`reactive-admin`. Skills in a cloned repository are not loaded automatically
unless the agent was started inside that repository, so open the `SKILL.md`
files directly. For the backend, read typiql-rs's `.claude/skills/typiql-api`.

## Layout

Clone per-form next to typical-admin and typical-admin-fabric; they import it
as `../per-form` and `../../../per-form`:

```
src/lib/per-form/
src/lib/typical-admin/
src/lib/typical-admin-fabric/
```

per-form itself only needs `react`.

## Usage

A schema is a plain object keyed by field name:

```tsx
import { useRef } from 'react';
import { Form } from './lib/typical-admin-fabric/lib';
import type { IFormRef } from './lib/typical-admin-fabric/lib/templates/Form';
import type { SchemaDefinition } from './lib/per-form';

const taskSchema: SchemaDefinition<any> = {
  title: { type: 'text', label: 'Title', required: true },
  status: {
    type: 'select', label: 'Status',
    options: [{ text: 'Open', value: 'open' }, { text: 'Done', value: 'done' }],
    defaultValue: 'open',
  },
  estimate: { type: 'number', label: 'Estimate (hours)', defaultNull: null },
};

function TaskForm({ onSave }: { onSave: (values: any) => void }) {
  const ref = useRef<IFormRef>(null);
  return (
    <>
      <Form ref={ref} name="task" form={taskSchema} initialValues={{ title: 'New task' }} />
      <button onClick={() => ref.current?.isValid && onSave(ref.current.submit())}>Save</button>
    </>
  );
}
```

- `initialValues` seeds the form once, at mount. To load a record, render the
  form after the data has arrived (or give it a `key` that changes).
- `submit()` returns the converted values (see below); `values` holds the raw
  ones. `onChange(name, { raw, clean, isValid, errors })` reports every change.
- To use per-form directly: `useForm({ schema, passedValues, converters })`
  returns `{ values, errors, dirty, touched, isValid, change, focus, reset, submit, schema }`;
  render with `<FormWrapper Template={YourTemplate} ... />`.

## Field properties per-form itself reads

| Property | Default | Meaning |
|---|---|---|
| `type` | required | Picks the template's control and the submit converter (`text`, `number`, `date` have built-in converters) |
| `label` | required | Shown by the template |
| `required` | `false` | Adds "This field is required" (fails when the value is falsy) |
| `validations` | `[]` | `{ test(form) => boolean, message: string \| (form) => string }[]` |
| `defaultValue` | `''` (`[]` for `list`) | Initial value when `initialValues` has none (or has `null`/`undefined`) |
| `defaultNull` | none | Submitted instead of `''` for an empty field |
| `converter` | none | `(key, values, defaultNull, dirty) => value`, overrides the type's converter at submit |
| `section`, `sectionCollapsed` | none | Groups fields into collapsible sections |

Every other property is passed through to the template (and, in Fabric.tsx,
on to the Fluent control). `display` is accepted but has no effect.

## Gotchas when the form feeds a GraphQL mutation

Verified in the source; all of them bite typiql apps:

- **Empty means `''`.** Unset fields, and `null`s from `initialValues`, become
  `''` and are submitted as `''`. GraphQL rejects `''` for `Int`, `Float`,
  `Boolean` and `ID`. Set `defaultNull: null` on optional non-string fields (and
  on optional strings, if you want `null` rather than `''` stored), and
  `defaultValue: false` on checkboxes.
- **The `number` converter only accepts `[0-9.]`.** `"-3"` is submitted as the
  string `"-3"` (rejected for `Int`/`Float`), and `"2.5"` is submitted as `2.5`
  even for an `Int` field. If negatives are possible, give the field its own
  converter:
  ```ts
  { type: 'number', label: 'Offset', defaultNull: null,
    converter: (key: string, values: any, defaultNull: any) =>
      values[key] === '' ? defaultNull : Number(values[key]) }
  ```
- **`required` fails on `0` and `false`.** It tests `!!value`. For a number
  where 0 is valid, use a validation instead:
  `validations: [{ test: (f) => f.qty !== '' && f.qty != null, message: 'Required' }]`.
  On a `list`, `required` is turned into `min: 1`.
- **Dates.** Fabric.tsx's `date` field stores a JS `Date` at local midnight;
  the `date` converter keeps it a `Date`, which serializes as a UTC timestamp
  (`2026-10-13T04:00:00.000Z` in UTC-4). If the backend wants a calendar date,
  convert it yourself: `converter: (k, v) => v[k] ? format(v[k], 'yyyy-MM-dd') : null`
  (`format` from `date-fns`).
- **Text is trimmed** on submit by the `text` converter; other unconverted
  types are submitted raw.
- **Field names must be plain keys.** `onChange` keeps only the part after the
  last `.`, so `address.city` becomes `city`. Use a `list` field (rows with
  their own nested form) for arrays of objects.

## Repeating rows: `list`

`{ type: 'list', label, itemSchema, rowKey, min, max, newRow, ... }` holds an
array of row objects; each row is its own nested form over `itemSchema`. See
`IListField` in `types.ts` for every option, and the `form-schema` skill.
`useRowCommit` is the same per-row commit primitive, for row editors that
can't be a `list`.
