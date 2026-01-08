# Hydrozoa

Hydrozoa is a proof-of-concept Java/Kotlin/WASM runtime for VEX V5.

However, its development is not ongoing due to technical limitations with the Java language. Java programs tend to have comparatively large executable sizes compared to other options, easily reaching several megabytes after including the standard library and interpreter. (For context, the absolute maximum you can upload to the brain is about 4mb). Additionally, as a result of Hydrozoa's architecture, Java bytecode is interpreted rather than compiled, leading to a roughly ~3x speed loss compared to native code. So, the vexide team is not focusing on Hydrozoa at this time.



## Summary

- [Pipeline](./pipeline.md)
