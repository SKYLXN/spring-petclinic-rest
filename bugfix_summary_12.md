# Patch for Issue #12

### Diagnosis

The root cause of the issue is that the code tries to insert a new Pet referencing an Owner by `ownerId` without first verifying that the Owner actually exists. This leads to a database-level foreign key constraint violation, which results in a 400 error instead of a 404 Not Found.

The correct approach is to check if the Owner with the given `ownerId` exists before attempting to add the Pet. If the Owner does not exist, the controller should return a 404 response.

Similarly, if the Pet being referenced (e.g., for update or other operations) does not exist, the controller should return a 404 error instead of a generic 400 or 500.

---

### Suggested Patch

Assuming a typical Spring MVC controller method for adding a Pet to an Owner, the patch involves:

- Fetching the Owner by `ownerId` before saving the Pet.
- Returning 404 if the Owner is not found.
- (Optionally) Handling Pet not found similarly in update/delete endpoints.

```diff
@@ -1,10 +1,28 @@
-// Controller method snippet (hypothetical)
-@PostMapping("/owners/{ownerId}/pets")
-public ResponseEntity<?> addPet(@PathVariable Long ownerId, @RequestBody Pet pet) {
-    pet.setOwner(new Owner(ownerId));
-    petRepository.save(pet);
-    return ResponseEntity.ok(pet);
-}
+@PostMapping("/owners/{ownerId}/pets")
+public ResponseEntity<?> addPet(@PathVariable Long ownerId, @RequestBody Pet pet) {
+    Optional<Owner> ownerOpt = ownerRepository.findById(ownerId);
+    if (!ownerOpt.isPresent()) {
+        return ResponseEntity.status(HttpStatus.NOT_FOUND)
+                .body("Owner with id " + ownerId + " not found");
+    }
+
+    Owner owner = ownerOpt.get();
+    pet.setOwner(owner);
+    Pet savedPet = petRepository.save(pet);
+    return ResponseEntity.status(HttpStatus.CREATED).body(savedPet);
+}
+
+// Similarly, for update or other operations involving Pets:
+@PutMapping("/owners/{ownerId}/pets/{petId}")
+public ResponseEntity<?> updatePet(@PathVariable Long ownerId, @PathVariable Long petId, @RequestBody Pet pet) {
+    if (!ownerRepository.existsById(ownerId)) {
+        return ResponseEntity.status(HttpStatus.NOT_FOUND)
+                .body("Owner with id " + ownerId + " not found");
+    }
+    Optional<Pet> petOpt = petRepository.findById(petId);
+    if (!petOpt.isPresent()) {
+        return ResponseEntity.status(HttpStatus.NOT_FOUND)
+                .body("Pet with id " + petId + " not found");
+    }
+    pet.setId(petId);
+    pet.setOwner(ownerRepository.getOne(ownerId));
+    Pet updatedPet = petRepository.save(pet);
+    return ResponseEntity.ok(updatedPet);
+}
```

---

### Test Recommendations

1. **Add Pet with Non-existent Owner**
   - Send POST `/owners/{nonExistingOwnerId}/pets`
   - Expect HTTP 404 with message "Owner with id X not found".

2. **Add Pet with Existing Owner**
   - Send POST `/owners/{existingOwnerId}/pets` with valid Pet data.
   - Expect HTTP 201 Created and Pet returned.

3. **Update Pet with Non-existent Owner**
   - Send PUT `/owners/{nonExistingOwnerId}/pets/{petId}`
   - Expect HTTP 404 for Owner not found.

4. **Update Pet with Non-existent Pet**
   - Send PUT `/owners/{existingOwnerId}/pets/{nonExistingPetId}`
   - Expect HTTP 404 for Pet not found.

5. **Update Pet with Existing

## Files modified
