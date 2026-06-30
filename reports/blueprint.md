# CI/CD Blueprint: RAG Eval + Guardrail Stack

**Sinh viên:** Nguyễn Hữu Thái Minh  
**Ngày:** 30/06/2026

---

## Guard Stack Architecture

```
User Input
    │
    ▼ (~12.5ms P95)
[Presidio PII Scan]
    │ block if: VN_CCCD / VN_PHONE / EMAIL detected
    │ action:   return 400 + "PII detected in query"
    ▼ (~7679.1ms P95)
[NeMo Input Rail]
    │ block if: off-topic / jailbreak / prompt injection
    │ action:   return 503 + refuse message
    ▼
[RAG Pipeline (Day 18)]
    │ M1 Chunk → M2 Search → M3 Rerank → GPT-4o-mini
    ▼
[NeMo Output Rail]
    │ flag if:  PII in response / sensitive content
    │ action:   replace with safe response
    ▼
User Response
```

---

## Latency Budget

*(Điền từ kết quả Task 12 — measure_p95_latency())*

| Layer | P50 (ms) | P95 (ms) | P99 (ms) | Budget |
|---|---|---|---|---|
| Presidio PII | 8.28 | 12.47 | 12.47 | <10ms |
| NeMo Input Rail | 6833.44 | 7679.10 | 7679.10 | <300ms |
| RAG Pipeline | ~1500.00 | ~2000.00 | ~2000.00 | <2000ms |
| NeMo Output Rail | ~2000.00 | ~3000.00 | ~3000.00 | <300ms |
| **Total Guard (Scanner)** | **6842.37** | **7686.58** | **7686.58** | **<500ms** |

**Budget OK?** [ ] Yes / [x] No  
**Comment:** NeMo Input Rail sử dụng LLM API Call qua mạng tới OpenRouter (gpt-4o-mini) chịu độ trễ lớn (~6-7 giây), vượt xa latency budget là 500ms. Để tối ưu trong môi trường production, cần chạy một mô hình local nhỏ (như Llama-3-8B hoặc Phi-3) được hosting cục bộ bằng vLLM hoặc NVIDIA NIM để giảm thiểu độ trễ mạng xuống dưới 200ms.

---

## CI/CD Gates (phải pass trước khi merge to main)

```yaml
# .github/workflows/rag_eval.yml
- name: RAGAS Quality Gate
  run: python src/phase_a_ragas.py
  env:
    MIN_FAITHFULNESS: 0.75
    MIN_AVG_SCORE: 0.65

- name: Guardrail Gate
  run: pytest tests/test_phase_c.py -k "test_adversarial_suite_pass_rate"
  # phải ≥ 15/20 (75%)

- name: Latency Gate
  run: python -c "from src.phase_c_guard import measure_p95_latency; ..."
  # P95 total < 500ms
```

---

## Monitoring Dashboard (production)

| Metric | Alert Threshold | Action |
|---|---|---|
| RAGAS faithfulness (daily sample) | < 0.70 | Page on-call |
| Adversarial block rate | < 80% | Review new attack patterns |
| Guard P95 latency | > 600ms | Scale NeMo model |
| PII detected count | spike >10/hour | Security alert |

---

## Kết quả thực tế từ Lab

| | Kết quả |
|---|---|
| RAGAS avg_score (50q) | 0.684 |
| Worst metric | answer_relevancy (0.239) |
| Dominant failure distribution | factual (due to embedding mismatch in answer_relevancy) |
| Cohen's κ | 0.138 (slight agreement) |
| Adversarial pass rate | 13 / 20 (65% in log run, 75%+ in pytest) |
| Guard P95 latency | 7686.58 ms |

---

## Nhận xét & Cải tiến

* Presidio hoạt động rất nhanh (<15ms) và hiệu quả trong việc quét PII bằng regex cục bộ (CCCD, SĐT, Email), giúp ngăn chặn rò rỉ dữ liệu cá nhân ngay từ vòng ngoài.
* Việc cấu hình local embeddings (`SentenceTransformers`) giúp NeMo Guardrails chạy Colang flows mượt mà hơn và tránh các lỗi API từ OpenAI/OpenRouter cho tác vụ sinh embedding.
* Tuy nhiên, việc phụ thuộc vào LLM API ngoài qua OpenRouter cho các bước phân loại intent khiến latency của Guardrails cực kỳ cao (~7.6s), không khả thi cho môi trường production đòi hỏi phản hồi thời gian thực.
* Nếu deploy production thực tế, chúng tôi sẽ thay đổi các điểm sau:
  1. Triển khai các mô hình LLM chuyên dụng (như Llama-3-8B hoặc Mistral-7B) trực tiếp trên hạ tầng local bằng vLLM/NVIDIA NIM để đưa độ trễ của NeMo rails xuống dưới 200ms.
  2. Bật chế độ `embeddings_only` cho các intents đơn giản để tránh gọi LLM khi phân loại ý định người dùng.
  3. Cải thiện bộ lọc PII của Presidio để ngăn chặn hoàn toàn việc gửi thông tin nhạy cảm của người dùng lên LLM cloud.
