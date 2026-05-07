# Bonus — Thread sweep

Model: `Llama-3.2-3B-Instruct-Q4_K_M.gguf`  ·  GPU layers: `99`

| threads | tg128 (tok/s) |
|---:|---:|
| 1 | 54.6 |
| 2 | 54.7 |
| 5 | 50.9 |
| 10 | 53.9 |
| 20 | 54.7 |

**Best**: `-t 20` at 54.7 tok/s.

Look at the curve. If it peaks around your **physical** core count and drops as you go higher, that's the memory-bandwidth ceiling: extra threads fight over the same memory channels and slow each other down.
