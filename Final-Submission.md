# Order Service App – Final Assignment Deliverable


## 1. Root Cause Analysis

### Error Observed

The application crashes when an order is placed with an unknown coupon code such as "BLACKFRIDAY". The logs and stack trace show that the failure occurs during discount calculation, before the order finishes successfully.

### Actual Runtime Evidence
From the runtime log:

INFO: Reserved 1 of MUG-010 (was 3, now 2)
  -> FAILED: java.lang.NullPointerException: Cannot invoke "com.rsl.orderservice.model.Coupon.getPercentOff()" because "coupon" is null
SEVERE: Scenario failed: B: order with unknown coupon BLACKFRIDAY

### Stack Trace Evidence
The exception is thrown in the discount service:

at com.rsl.orderservice.service.DiscountService.discountCents(DiscountService.java:52)
at com.rsl.orderservice.service.OrderService.placeOrder(OrderService.java:78)
at com.rsl.orderservice.App.lambda$main$1(App.java:63)
at com.rsl.orderservice.App.runScenario(App.java:79)
at com.rsl.orderservice.App.main(App.java:62)


### Exact Throwing Line
The exception is thrown at the first dereference of the coupon object:

log.info("Applying coupon '" + couponCode + "' -> " + coupon.getPercentOff() + "%");

This occurs inside DiscountService.discountCents() and is the first place where execution fails because coupon is null.

### Root Cause
The root cause is that DiscountService.discountCents() calls findByCode() and immediately dereferences the returned Coupon without checking whether the repository returned null.

The repository behavior is documented and implemented as:

java
/** Returns the coupon, or {@code null} if the code is unknown. */
public Coupon findByCode(String code) {
    return coupons.get(code);
}


This means that for an unregistered coupon like BLACKFRIDAY, findByCode() returns null. The service then calls coupon.getPercentOff(), which throws the NullPointerException.

### Supporting Code Evidence

if (couponCode != null && !couponCode.isBlank()) {
    Coupon coupon = couponRepository.findByCode(couponCode);
    log.info("Applying coupon '" + couponCode + "' -> " + coupon.getPercentOff() + "%");
    percent += coupon.getPercentOff();
}


This matches the observed behavior exactly: the bug is a missing null-check at the boundary between the repository contract and the service logic.


## 2. AI Interaction Transcript


### Step 1 – Reproduce the runtime failure

Prompt used:

I am working on the order-service-app project. Please read README.md and the project code, then reproduce the runtime failure by running the application and the tests. Identify which scenario fails, which test fails, and where the failure is logged. Also point me to the exact error written in logs/app.log.


AI response:
- The app fails in Scenario B for the unknown coupon BLACKFRIDAY.
- One unit test fails.
- The failure is recorded in logs/app.log.

### Step 2 – Retrieve runtime log with the right MCP server

Prompt used:

GitHub MCP cannot access the local runtime log file because it only reads repository data. Please identify the correct MCP server for local filesystem access and use it to read app.log. Explain why GitHub MCP is not correct here and show the retrieved runtime log contents.

AI response:
- GitHub MCP cannot inspect local runtime files.
- A local file-access MCP was required for this task.
- The runtime log was retrieved using the Desktop Commander MCP local filesystem access, which can read the generated application log file from the workspace.
- The retrieved log confirmed the unknown-coupon failure and captured the exact NullPointerException stack trace.

Runtime log retrieved via local MCP access:

  -> FAILED: java.lang.NullPointerException: Cannot invoke "com.rsl.orderservice.model.Coupon.getPercentOff()" because "coupon" is null

### Step 3 – Create the issue on GitHub

Prompt used:

Please create a GitHub issue in this repository titled with the failure observed in the app. The issue body should describe the exact observed behavior and steps to reproduce, without guessing at the cause or fix. Include the runtime exception, affected scenario, and log evidence.


AI response:
- An issue was created describing the runtime failure and steps to reproduce.
- It documented the observed behavior but did not speculate on the fix.

### Step 4 – Analyze the stack trace

Prompt used:

analyze the stack trace from the crash in app.log. State the exact line where the error was thrown and the exact call path from application entry to that line. Include the class name, method name, and line numbers.


AI response:
- Exact throw site: DiscountService.java:52
- Call path: App.main() → App.runScenario() → App.lambda$main$1() → OrderService.placeOrder() → DiscountService.discountCents()

### Step 5 – Separate issue and root cause

Prompt used:

give me the issue observed in the application, then return the root cause separately and cite the specific code behavior that supports it. Use the repository code, the stack trace, and the log as evidence


AI response:
- Issue: application crashes with a NullPointerException when an unknown coupon is used.
- Root cause: repository returns null and the service dereferences it without validation.

### Step 6 – Explore fix options

Prompt used:

suggest one or more possible fixes for this bug. For each option, explain the reasoning and cons. Then identify the recommended fix and explain why it is the best solution for this application

AI response:
- Multiple fix options were discussed.
- Recommended option: guard the null coupon case and either ignore or fail clearly with a descriptive exception.

### Step 7 – Validate the recommended fix before applying it

Prompt used:

Before changing the code according to the recommended 2nd option, explain fix and how it addresses the root cause. Confirm that this fix prevents the crash when an unknown coupon code is passed and preserves the expected behavior of valid coupons.


AI response:
- A null-check prevents the dereference.
- Valid coupons still follow the same logic.
- Unknown coupons are handled safely and predictably.

### Step 8 – Branch, commit, and PR workflow

Prompt used:


diff --git a/src/main/java/com/rsl/orderservice/service/DiscountService.java b/src/main/java/com/rsl/orderservice/service/DiscountService.java
index 6fe6138..ead3dd5 100644
--- a/src/main/java/com/rsl/orderservice/service/DiscountService.java
+++ b/src/main/java/com/rsl/orderservice/service/DiscountService.java
@@ -1,12 +1,12 @@
 package com.rsl.orderservice.service;
 
+import java.util.logging.Logger;
+
 import com.rsl.orderservice.model.Coupon;
 import com.rsl.orderservice.model.Customer;
 import com.rsl.orderservice.repository.CouponRepository;
 import com.rsl.orderservice.util.AppLogger;
 
-import java.util.logging.Logger;
-
 /**
  * Decides the discount percentage that applies to an order, based on
  * membership status and an optional coupon code.
@@ -49,6 +49,9 @@ public class DiscountService {
 
         if (couponCode != null && !couponCode.isBlank()) {
             Coupon coupon = couponRepository.findByCode(couponCode);
+            if (coupon == null) {
+                throw new IllegalArgumentException("Unknown coupon code: " + couponCode);
+            }
             log.info("Applying coupon '" + couponCode + "' -> " + coupon.getPercentOff() + "%");
             percent += coupon.getPercentOff();
         }

Create a branch named nullPointerException-in-DiscountService. Commit the fix with a clear message referencing issue and the root cause. Then open a pull request with a proper description that includes the problem, the fix, and verification details.

AI response:
- The branch and fix were prepared.
- The fix was committed locally with an issue reference.
- Created PR with recommended approach. 

## 3. Final Resolution

### Fix Applied
The fix adds a null guard before dereferencing the coupon returned by CouponRepository.findByCode().

java
if (couponCode != null && !couponCode.isBlank()) {
    Coupon coupon = couponRepository.findByCode(couponCode);
    if (coupon == null) {
        throw new IllegalArgumentException("Unknown coupon code: " + couponCode);
    }
    log.info("Applying coupon '" + couponCode + "' -> " + coupon.getPercentOff() + "%");
    percent += coupon.getPercentOff();
}


### Why this fix was selected
- It specifically addresses the real root cause.
- It prevents a NullPointerException when a coupon code is missing.
- It leaves valid coupon behavior unchanged.

### Verification Summary
The project was tested with:
mvn test

Observed result from the test run:

Tests run: 12, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS


### Before / After Behavior
Before the fix:
- Unknown coupon BLACKFRIDAY caused a runtime crash.
- Scenario B failed.

After the fix:
- The null case is handled explicitly.
- Valid coupon behavior remains correct.
- The test suite passes without regression.


## 4. GitHub Issue and Pull Request Details

### Issue
- Issue reference: https://github.com/rsllearning/order-service-app/issues/1

### Pull Request
PR link: https://github.com/rsllearning/order-service-app/pull/2.

## 5. Video Link

https://drive.google.com/file/d/1gJXlt6eHsdtC-X7Uu3O10acVLmTw9XCa/view?usp=sharing
