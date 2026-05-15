# Decision Tree — <Solution Name>

> Replace this placeholder with conditional logic the agent uses to map user answers to component choices.

## How to use

For each user input, the agent walks the relevant decision tree below to select stacks, configurations, and patterns.

## Decision: <Aspect 1>

```
Q: <question>?
│
├─ <answer A> → <component / pattern>
├─ <answer B> → <component / pattern>
└─ <answer C> → <component / pattern>
```

## Decision: <Aspect 2>

```
Q: <question>?
│
├─ <answer A> → <component / pattern>
└─ <answer B> → <component / pattern>
```

## Cost / region considerations

When the user's region or budget conflicts with a recommended choice, downgrade to the closest alternative and document the trade-off in the design summary.
