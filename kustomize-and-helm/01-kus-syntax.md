# Kustomization 語法彙整

## 名詞解釋

* Transformer：使用 **kustomize 的語法**來修改樣本 yaml。設定方式可分兩種：
  1. Inline：直接寫在 kustomization.yml 中。
  2. configurations：寫法同 Inline，只不過放在另外的 yaml 檔案中，再透過 `configurations` 引入至 kustomization.yml。

* patches：使用 **JSON6902 Patch** 或 **Strategic Merge Patch** 來修改樣本 yaml。設定方式可分兩種：
  1. Inline：直接寫在 kustomization.yml 中。
  2. patches：寫法同 Inline，只不過放在另外的 yaml 檔案中，再透過 `patches.path` 引入至 kustomization.yml。

> 兩者能達到的功能基本相同，筆者