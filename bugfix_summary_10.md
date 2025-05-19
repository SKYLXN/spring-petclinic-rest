# Patch for Issue #10

### Diagnosis

The root cause of the `400 Bad Request` response is the `equals()` check between `pet.getOwner()` and `owner` in the `getOwnersPet` method. Since the `Owner` entity class does not override `equals()` and `hashCode()`, the default identity-based equality check is used, which returns `false` even if the two `Owner` instances represent the same database record with the same `id`.

This causes the controller to wrongly conclude that the pet does not belong to the owner and return a `400 Bad Request`.

### Suggested Patch

The best practice is to override `equals()` and `hashCode()` in entity classes based on their identifier (`id`), or at least compare their IDs explicitly in the controller method.

Since modifying entities might have broader implications, the quickest and safest fix here is to replace the `equals()` check with an ID comparison in `OwnerRestController#getOwnersPet`.

```diff
@@ -11,10 +11,14 @@
-        if (!pet.getOwner().equals(owner)) {
-            return new ResponseEntity<>(HttpStatus.BAD_REQUEST);
+        Owner petOwner = pet.getOwner();
+        if (petOwner == null || !petOwner.getId().equals(owner.getId())) {
+            return new ResponseEntity<>(HttpStatus.BAD_REQUEST);
         } else {
-            return new ResponseEntity<>(petMapper.toPetDto(pet), HttpStatus.OK);
+            return new ResponseEntity<>(petMapper.toPetDto(pet), HttpStatus.OK);
         }
```

Optionally, for a more robust long-term fix, add `equals()` and `hashCode()` to the `Owner` entity based on `id`:

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Owner)) return false;
    Owner owner = (Owner) o;
    return id != null && id.equals(owner.id);
}

@Override
public int hashCode() {
    return 31;
}
```

Similarly for `Pet`. But this requires careful consideration of entity lifecycle and Hibernate proxying.

### Test Recommendations

- Write integration tests for the endpoint `GET /api/owners/{ownerId}/pets/{petId}` to verify:
  - When the pet belongs to the owner, the endpoint returns 200 OK with correct pet data.
  - When the pet does not belong to the owner, the endpoint returns 400 Bad Request.
  - When either owner or pet does not exist, the endpoint returns 404 Not Found.
- Test with different owner and pet IDs to cover edge cases.
- If `equals()` and `hashCode()` are added to entities, test entity comparisons and collections behavior.

## Files modified
