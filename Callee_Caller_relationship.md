Calling conventions often use terms such as callee and caller:

* The caller is the function that makes the call, and the callee is the function being called.
* Consider a function foo(), being called from main():
<img width="885" height="270" alt="image" src="https://github.com/user-attachments/assets/726d9e67-35c9-4f4d-90ad-4127649b80c1" />

* In this example, main() is the caller — it contains the line a = foo(b, c); — and foo() is the callee, since it's the one being invoked by main().
* This distinction matters because the PCS(Procedure Call Standard in AARCH64)/CC(Calling Convention) is essentially a contract between the two roles.
* As long as both sides agrees it, the caller and callee can be written completely independently — different files, different languages, even produced by different compilers — and still interoperate correctly.
* Neither one needs to know how the other is implemented internally, only what it's promised regarding registers and the stack.
* In the Image, before foo() is called, the caller (main()) is responsible for loading the arguments into the parameter registers — here b goes into W0 and c into W1, since both are 32-bit values. Once that's set up, main() hands off control to foo() with a BL (branch with link) instruction.
* BL does two things at once: it jumps to foo's first instruction, and it writes the address of the instruction right after the BL into LR (X30) — that's the "return address" highlighted in the diagram.
<img width="1030" height="366" alt="image" src="https://github.com/user-attachments/assets/e6985f2c-c828-4b7b-a8cf-39e1c270fd91" />

* From here, foo() is in charge as the callee. It can freely use registers X0–X15 (plus the IP0/IP1 scratch registers) for whatever it needs, without restoring them afterward — these are the "corruptible" registers as per ABI convention.
* But if foo() needs to use any of the callee-saved registers, X19 through X28, it has to save their original values to the stack first and restore them before returning.
* That's what lets main() trust those registers still hold whatever it left in them, even after the call. LR itself effectively works the same way: if foo() called another function internally, it would have to save its own LR first, since that nested BL would overwrite it — though in this simple example foo() doesn't call anything else, so LR stays untouched.
* Before returning, foo() places its result in X0 (or X0–X1, for larger return values) and executes RET rather than a plain branch. RET jumps to whatever address is currently sitting in LR, landing back on the instruction right after the original BL foo in main() — the lower arrow in both your images — and main() continues from there with the result now sitting in X0.
* So the caller's job is: set up the inputs, hand off control, and trust that anything marked callee-saved survives the call. The callee's job is: use the inputs, do whatever internal work it needs (saving and restoring any registers it borrows), and hand back a result through the agreed-upon channel. 
