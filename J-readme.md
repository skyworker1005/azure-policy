
이 저장소는 **Terraform 코드가 아니라**, Microsoft의 **Azure Policy 샘플(JSON 정의)** 저장소입니다. 구조와 Terraform 연동 방법을 정리했습니다.

---

## 1. 저장소 성격

- **Azure/azure-policy**: Azure **Built-in 정책** 정의를 JSON으로 제공하는 공식 샘플 저장소
- **Terraform(.tf) 파일은 없음** → 정책 정의는 모두 JSON (`azurepolicy.json`, `azurepolicy.rules.json`, `azurepolicy.parameters.json`)

---

## 2. 정책 파일 구조

각 샘플은 보통 다음 3개 파일로 구성됩니다.

| 파일 | 역할 |
|------|------|
| **azurepolicy.json** | 정책 전체 정의 (메타데이터 + parameters + policyRule) |
| **azurepolicy.rules.json** | `policyRule`만 분리한 규칙 (if/then) |
| **azurepolicy.parameters.json** | `parameters` 스키마만 분리 |

### 2.1 정책 정의(azurepolicy.json) 공통 구조

```json
{
  "properties": {
    "displayName": "표시 이름",
    "policyType": "BuiltIn",
    "mode": "Indexed",           // Indexed = RG, Subscription, 리소스 / All = 모든 리소스 타입
    "description": "설명",
    "parameters": { ... },      // 선택적 파라미터
    "policyRule": {
      "if": { ... },            // 조건
      "then": { "effect": "..." }  // 효과 (Deny, Audit, AuditIfNotExists 등)
    }
  },
  "type": "Microsoft.Authorization/policyDefinitions",
  "name": "GUID"
}
```

- **Built-in** 정책은 `id`/`name`에 GUID가 들어가고, 커스텀 정책은 배포 시 이름/경로를 정함.

---

## 3. 샘플별 정책 유형

### 3.1 Allowed locations (허용 지역)

- **효과**: `Deny`
- **역할**: 지정한 지역 외 리소스 배포 차단
- **조건**: `location`이 `listOfAllowedLocations`에 없고, `global`이 아니며, 타입이 `Microsoft.AzureActiveDirectory/b2cDirectories`가 아닐 때 Deny

### 3.2 Require tag and its value (태그 필수)

- **효과**: `deny`
- **역할**: 특정 태그 이름·값이 없으면 리소스 생성/수정 거부
- **조건**: `tags[tagName]` ≠ `tagValue` 이면 Deny

### 3.3 Allowed resource types (허용 리소스 타입)

- **효과**: `deny`
- **역할**: 허용 목록에 없는 리소스 타입 배포 차단
- **조건**: `type`이 `listOfResourceTypesAllowed`에 없으면 Deny

### 3.4 DeployIfNotExists (Network Watcher 예시)

- **효과**: `deployIfNotExists`
- **역할**: VNet 생성 시 해당 지역에 Network Watcher가 없으면 자동 배포
- **구성**: `existenceCondition`, `roleDefinitionIds`, ARM `deployment` 템플릿 포함

---

## 4. Terraform으로 이 정책 배포하기

이 저장소의 JSON 정의를 그대로 쓰려면 Terraform에서 **정책 정의 JSON을 읽어서** `azurerm_policy_definition`에 넘기면 됩니다.

### 4.1 커스텀 정책 정의 등록

```hcl
# azurerm_policy_definition 리소스 예시
resource "azurerm_policy_definition" "allowed_locations" {
  name         = "allowed-locations-custom"
  policy_type  = "Custom"
  mode         = "Indexed"
  display_name = "Allowed locations (from sample)"
  description  = "Restrict resource deployment to allowed locations."

  policy_rule = file("${path.module}/samples/built-in-policy/allowed-locations/azurepolicy.rules.json")
  parameters  = file("${path.module}/samples/built-in-policy/allowed-locations/azurepolicy.parameters.json")
}
```

- `policy_rule`: `azurepolicy.rules.json` 내용  
- `parameters`: `azurepolicy.parameters.json` 내용  
- Built-in을 그대로 쓰지 않고 **커스텀으로 재정의**하는 패턴입니다.

### 4.2 정책 할당(Assignment)

```hcl
resource "azurerm_resource_policy_assignment" "allowed_locations" {
  name                 = "allowed-locations-assignment"
  resource_id           = azurerm_resource_group.example.id
  policy_definition_id = azurerm_policy_definition.allowed_locations.id
  parameters = jsonencode({
    listOfAllowedLocations = {
      value = ["koreacentral", "koreasouth"]
    }
  })
}
```

- 구독/관리 그룹 단위라면 `azurerm_subscription_policy_assignment`, `azurerm_management_group_policy_assignment` 사용.

### 4.3 Initiative(정책 세트) 예시

여러 정책을 묶어서 Initiative로 배포할 때:

```hcl
resource "azurerm_policy_set_definition" "governance" {
  name         = "governance-initiative"
  policy_type  = "Custom"
  display_name = "Governance initiative"
  description  = "Combined policy set"

  parameters = file("${path.module}/parameters.json")

  policy_definition_reference {
    policy_definition_id = azurerm_policy_definition.allowed_locations.id
    parameter_values     = jsonencode({ listOfAllowedLocations = { value = ["koreacentral"] } })
  }
  policy_definition_reference {
    policy_definition_id = azurerm_policy_definition.require_tag.id
    parameter_values     = jsonencode({ tagName = { value = "Environment" }, tagValue = { value = "Production" } })
  }
}
```

---

## 5. 요약

| 항목 | 내용 |
|------|------|
| **저장소 내용** | Azure Policy Built-in 샘플 **JSON** (Terraform 코드 없음) |
| **정책 형식** | `azurepolicy.json` + rules/parameters 분리 파일 |
| **Terraform 연동** | `file()`로 JSON 읽어 `azurerm_policy_definition`의 `policy_rule`/`parameters`에 전달 후, `azurerm_*_policy_assignment`로 할당 |

Terraform으로 “이 저장소의 특정 샘플을 그대로 배포하는” 예제를 원하면, 원하는 샘플 폴더 이름(예: `allowed-locations`, `enforce-tag-value`)을 알려주시면 해당 경로 기준으로 완전한 `.tf` 예제를 적어드리겠습니다.