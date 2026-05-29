## New: Fix for dangerouslySetInnerHTML biome warning

When using `dangerouslySetInnerHTML`, add biome-ignore comment on the line BEFORE the element that uses the prop:

```typescript
// biome-ignore lint/security/noDangerouslySetInnerHtml: lesson content from trusted API
<div dangerouslySetInnerHTML={{ __html: cleanSspmTags(content) }} />
```

The ignore comment must be on the line immediately before the element, not on the same line as the prop.