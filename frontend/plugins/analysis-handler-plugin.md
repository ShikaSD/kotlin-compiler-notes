# Analysis Handler

- Handling analysis.
- `doAnalysis` to launch it and `analysisCompleted` to hook into compiler process.
- `AnalysisResult` to either continue with codegen, error, or repeat analysis.
- KSP is built on top of it.