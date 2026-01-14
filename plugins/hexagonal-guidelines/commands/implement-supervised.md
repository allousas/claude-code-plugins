---
allowed-tools: *
argument-hint: [feature] OR [workflow] [feature]
description: Implements feature with supervised workflow. Defaults to supervised-tdd-workflow. Can also use supervised-upfront-workflow for design-first approach.
---

{{#if $2}}
Implement this feature using the $1 skill: $2.
{{else}}
Implement this feature using the supervised-tdd-workflow skill: $1.
{{/if}}