# 希望啟用的 clang-tidy 檢查

以下檔案包含專案希望啟用、但目前因工程上的困難，或因有其他更高優先級的
重構工作而尚未啟用的檢查與設定。大家普遍認同下列檢查理論上_應該_是有幫助的，
但尚未在整個專案中實際啟用它們。

```yaml
Checks: "-*, readability-function-size,
  readability-function-cognitive-complexity
  "

CheckOptions:
  - key: readability-function-size.LineThreshold
    value: 60 # [1]
  - key: readability-function-size.ParameterThreshold
    value: 6 # [2]
  - key: readability-function-cognitive-complexity.Threshold
    value: 25 # [3]


# [1] https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#f3-keep-functions-short-and-simple
# [2] https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#f2-a-function-should-perform-a-single-logical-operation
# [3] https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#f3-keep-functions-short-and-simple
# However cognitive complexity != cyclomatic complexity. Therefore using the clang-tidy default value,
# as cyclomatic complexity seems to not be implemented in clang-tidy.

# [1],[2],[3] do not have to be enforced or applied project-wide yet.
```
