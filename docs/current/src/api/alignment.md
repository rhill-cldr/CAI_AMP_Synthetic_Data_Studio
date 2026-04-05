# Alignment Routes

One endpoint handles model alignment data generation, producing both DPO (Direct Preference Optimization) and KTO (Kahneman-Tversky Optimization) formatted outputs from seed data.

---

## POST /model/alignment

Generate model alignment data in DPO and KTO formats.

### Parameters

This endpoint accepts two JSON request bodies plus query parameters.

| Parameter | Type | Source | Description |
|---|---|---|---|
| synthesis\_request | `SynthesisRequest` | body | Parameters for generating alternate completions |
| evaluation\_request | `EvaluationRequest` | body | Parameters for scoring both responses |
| job\_name | string? | query | Optional job identifier for tracking |
| is\_demo | bool | query (default=`true`) | Execution mode |

### Pipeline Flow

1. **Generate alternates:** `ModelAlignment.alternate_result()` reads seed data from `synthesis_request.input_path`, extracts items using `output_key`/`output_value`, and generates alternate completions via `SynthesisLegacyService.process_single_input()` with `ThreadPoolExecutor` (`max_workers=5`).
2. **Score originals:** `ModelAlignment.evaluate_generation()` scores the original completions using `EvaluatorLegacyService.evaluate_single_pair()` with `ThreadPoolExecutor` (`max_workers=4`).
3. **Score alternates:** Same evaluation step applied to alternate completions.
4. **Build DPO pairs:** Compare scores. Higher-scored response becomes `chosen`, lower becomes `rejected`.
5. **Build KTO entries:** Flatten DPO pairs into binary preference entries (`Label=True` for chosen, `Label=False` for rejected).

### Response

```json
{
  "status": "success",
  "dpo": [
    {
      "instruction": "...",
      "chosen_response": "...",
      "rejected_response": "...",
      "chosen_rating": 4.0,
      "rejected_rating": 2.0
    }
  ],
  "kto": [
    {
      "Prompt": "...",
      "Completion": "...",
      "Label": true,
      "Rating": 4.0
    },
    {
      "Prompt": "...",
      "Completion": "...",
      "Label": false,
      "Rating": 2.0
    }
  ]
}
```

---

## DPO Output Schema

| Field | Type | Description |
|---|---|---|
| instruction | string | The input prompt |
| chosen\_response | string | Higher-scored completion |
| rejected\_response | string | Lower-scored completion |
| chosen\_rating | float | Score of chosen response |
| rejected\_rating | float | Score of rejected response |

## KTO Output Schema

| Field | Type | Description |
|---|---|---|
| Prompt | string | The input prompt |
| Completion | string | The response text |
| Label | bool | `true`=chosen, `false`=rejected |
| Rating | float | Evaluation score |

---

## Error Handling

| Condition | Status | Error |
|---|---|---|
| `APIError` during alignment | 400 | Error detail string |
| Unexpected exception | 500 | `"Internal server error: {detail}"` |
