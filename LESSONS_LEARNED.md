# Multi-Agent Orchestration di Domain Linear: Catatan Eksplorasi

> **Last updated:** 2026-05-11. Project di-finalize sebagai eksplorasi pattern; tidak dikembangkan lebih lanjut sebagai produk.
>
> **Catatan revisi 2026-05-11:** section 3.3 + 3.4 + 4 di-revisi setelah debugging Minimax di project lain (HADE) yang nge-clarify *layer* di mana tool-use failure sebenernya terjadi. Lihat juga `~/Documents/jk/Green-Hade/learn/tool-use-protocol-emission.md`.

## 1. Konteks

Project ini dibangun sebagai eksplorasi multi-agent orchestration pattern via LangGraph supervisor. Fokus utamanya adalah pattern itu sendiri — bagaimana orchestrator merute ke specialist, bagaimana state dibagi antar agent, bagaimana coordination dilakukan dalam satu turn. Domain health & nutrition tracking dipilih sebagai *vehicle*, bukan tujuan: domain butuh terlihat masuk akal untuk multi-agent supaya implementasi tetap realistis, tapi nilai akhir project terletak pada apa yang dipelajari tentang pattern.

Implementasi: empat specialist agent (assessment, planning, tracking, intervention) di bawah satu orchestrator LLM yang me-route lewat tool call `transfer_to_<agent>`. Stack: LangGraph supervisor, multi-provider LLM (default Claude, eksperimen dengan Minimax-M2.7), PostgreSQL (data + checkpointer), Qdrant + fastembed (RAG nutrition), Next.js frontend.

Setelah implementasi penuh, testing end-to-end via web frontend, dan eval kuantitatif via LangSmith + DeepEval (15 case, lima metrik: routing_accuracy, task_completion, response_quality, tool_correctness, plus no_loop check), finding utamanya bukan tentang efektifitas multi-agent secara umum — melainkan tentang **pattern-domain fit**. Pattern ini punya nilai nyata di domain dengan karakteristik tertentu (paralelisme sub-task, expertise yang divergen, routing yang genuinely butuh judgment). Health tracking flow yang linear (assess → plan → track → intervene) tidak punya karakteristik itu. Hasilnya: supervisor pattern over-engineered untuk domain ini, dan biaya over-engineering tersebut bisa diukur.

Catatan ini merangkum bukti dari eval, akar masalah arsitektural yang muncul, apa yang akan dilakukan berbeda, dan kondisi di mana pattern ini sebenarnya tepat.

## 2. Hasil Eval — Trade-off yang Terkuantifikasi

Setelah dua iterasi prompt fix (orchestrator routing rules + scope rule per agent), eval batch terakhir (Run A) dibandingkan dengan baseline pra-fix (Run B):

| Metrik | Run A | Run B | Δ |
|---|---|---|---|
| routing_accuracy | 0.733 | 0.533 | **+0.200** |
| response_quality | 0.671 | 0.593 | +0.078 |
| tool_correctness | 0.456 | 0.433 | +0.023 |
| task_completion | 0.689 | 0.920 | **-0.231** |
| no_loop | 0.786 | 1.000 | **-0.214** |

Total token batch: 89K → 141K (+58%). Latency p50 33s → 43s (+30%). p99 85s → 130s (+53%).

Routing accuracy naik signifikan, sesuai intent fix. Tapi task_completion turun 25%, no_loop turun 21%, biaya naik 58%. Pola ini bukan kebetulan — perbaikan di satu titik (orchestrator routing) memicu regresi di titik lain (penyelesaian task multi-handoff, deteksi loop). Ini gejala arsitektural, bukan bug agent yang bisa di-patch dengan satu prompt rule lagi.

## 3. Akar Masalah Arsitektural

Tiga isu muncul berulang sepanjang eksplorasi.

### 3.1 Routing decision deterministik, tapi dijalankan oleh LLM

Dari 15 case eval, mayoritas intent user bisa diklasifikasi dengan keyword sederhana: "log makan" → tracking, "kasih plan" → planning, "halo aku baru" + setup keyword → assessment. Memakai supervisor LLM sebesar M2.7 sebagai router berarti membayar pajak fleksibilitas yang tidak terpakai. Classifier kecil (Haiku dengan structured output, atau bahkan regex untuk keyword) akan lebih konsisten dan jauh lebih murah.

### 3.2 State antar-agent implicit via conversation history

Specialist saling "berbicara" lewat message history. Setiap agent harus parse ulang user data dari teks bebas tiap turn. Pada case 13 (`"umur 25, berat 70kg, sedentary, mau turun 5kg dalam 2 bulan — kasih plan diet"`), planning agent awalnya menghasilkan plan generik karena prompt mengandung frasa `"Base it on the user's assessment data"`. Model menafsirkan literal: tunggu data dari assessment, padahal user sudah menulis seluruh data di message. Solusi prompt-level mungkin saja, tapi seluruh kelas masalah ini hilang kalau profile disimpan sebagai struktur typed di PostgreSQL dengan field eksplisit `age, weight, activity, goals` yang dibaca/ditulis lewat tool, bukan disimpulkan dari teks.

### 3.3 Patch on patch pada prompt orchestrator

Selama project, ORCHESTRATOR_PROMPT bertambah secara akumulatif: STOP RULES → COORDINATION EXCEPTION → ANTI-OVERSTEP → ANTI-FABRICATION → AMBIGUOUS INPUT → OUTPUT RULES. Tiap rule menutup satu mode kegagalan, tapi menambah area yang harus dijaga konsistensinya. Cross-model test (Minimax M2.7 vs Claude Haiku 4.5, query case 13) memperlihatkan batas pendekatan ini, dan kedua model gagal di **layer yang berbeda**:

- **Minimax**: skip `collect_health_data` — reasoning correct, **tool emission gagal**. Model paham harus call tool, argumen benar, tapi gagal output `tool_use` block sesuai Anthropic API spec. Ini failure di *protocol layer*, bukan reasoning layer.
- **Haiku**: emit tool_use OK technically, tapi **memilih** untuk ask user (tinggi badan) daripada proceed. Failure di *reasoning/interpretation layer* — over-cautious interpretasi `SCOPE RULE`.

Implikasinya: prompt yang sama tidak konvergen bukan karena salah satu model "lebih jelek", tapi karena prompt-engineering tidak bisa simultaneously menutup dua failure mode yang independen. Setiap aturan baru di prompt menutup satu mode di satu model, kadang membuka mode lain di model berbeda.

### 3.4 Decomposition failure per provider

Setelah trace inspection lintas case, attribution failure kira-kira:

| Failure source | Minimax M2.7 | Claude Haiku 4.5 |
|---|---|---|
| Protocol emission (tool_use format gagal di-emit) | ~30-40% | ~0% |
| Reasoning/interpretation (over-cautious, salah rule lookup) | ~30-40% | ~50-60% |
| Prompt complexity (orchestrator rules clash) | ~20-30% | ~30-40% |
| State management implicit (data hilang antar handoff) | ~10-20% | ~10-20% |

Catatan: angka kasar dari sampling case eval, bukan formal attribution study.

Yang penting bukan angkanya, tapi shape distribusi-nya: **Minimax punya satu extra failure source (protocol emission) yang Haiku tidak punya, tapi ketiga source lain dipakai bareng**. Itu sebabnya:

- Switch model saja tidak akan converge ke "100% reliable" — failure source 2-4 model-agnostic, bertahan di provider mana pun.
- Patch prompt iteratif gagal: tiap patch hanya nyentuh satu source di satu model, kadang nge-amplify source lain di model berbeda.
- Solusi struktural (section 4) harus mengurangi *dependence pada ketiga skill*, bukan pilih provider yang terbaik di salah satunya.

## 4. Apa yang Akan Dilakukan Berbeda

Pendekatan hybrid: deterministic outer + LLM specialist single-call.

```
chat → intent_classifier (Haiku, structured output, 1 call)
     → switch
        case setup_intent     → assessment_agent
        case plan_request     → ensure_profile_complete → planning_agent
        case meal_log         → tracking_agent
        case adherence_check  → intervention_agent
```

Konsekuensi konkret:

- Routing keluar dari LLM judgment. Debug = log statement, bukan trace inspection di LangSmith.
- State explicit di PostgreSQL (typed profile + plan + meal log object), bukan disimpulkan dari message history.
- **Dependency pada protocol emission dieliminasi**: orchestrator pakai keyword router (regex/classifier kecil), specialist pakai single structured call dengan Pydantic schema yang divalidasi di client. Tool_use block-emission tidak lagi jadi failure source — model tinggal output JSON di text block, parse manual. Implikasi: provider tool-marginal seperti Minimax M2.7 jadi viable lagi untuk role yang tidak butuh agentic loop.
- Tool schema typed: `create_health_plan(age: int, weight: float, activity: ActivityLevel, goal: HealthGoal, target_kg: float, timeline_weeks: int)`. Memaksa structured extraction; tidak ada `plan_summary: str` free-form yang membiarkan model improvisasi.
- Tidak ada multi-handoff dalam satu turn. Loop class hilang sepenuhnya, bukan dideteksi setelah terjadi.

Estimasi kasar: 40-50% reduksi token per turn, latency turun ke single-LLM-call territory, jumlah prompt rule yang perlu dijaga turun drastis.

Yang hilang: fleksibilitas multi-agent dynamic routing untuk intent yang tidak bisa diklasifikasi. Untuk health tracking, fleksibilitas itu memang tidak terpakai.

## 5. Kapan Supervisor Pattern Justru Tepat

Pattern ini bukan tanpa tempat. Nilai nyatanya muncul ketika:

- **Sub-task paralel independen** — research assistant yang query banyak sumber lalu sintesis, dengan setiap source-handler punya prompt dan knowledge base berbeda
- **Setup adversarial** — debate simulator, red-team vs blue-team, proof-by-contradiction
- **Expertise divergen pada input yang sama** — review dokumen dari sudut legal + finance + medical, tiap aspek butuh model/prompt/RAG berbeda
- **Routing decision genuinely butuh judgment** — input ambigu, intent tidak bisa dipetakan ke keyword atau pattern

Health intelligence flow tidak punya karakteristik di atas. Sequence-nya linear (assess → plan → track → intervene), routing 80% bisa di-keyword-match, expertise tiap agent overlap signifikan (semua adalah "asisten gizi" dengan persona berbeda).

## 6. Penutup

Karena tujuan eksplorasi adalah pattern, bukan produk, finding terpenting project ini bersifat meta: **pemilihan pattern arsitektur bukan kosmetik, dan domain-pattern fit bisa diukur**. Routing accuracy yang naik bersamaan dengan task_completion yang turun, dengan biaya per turn yang naik 58%, adalah bukti konkret. Domain yang lebih sederhana dari pattern yang dipakai akan menanggung pajak fleksibilitas yang tidak terpakai — dan pajak itu muncul di metrik.

Eksplorasi multi-agent berikutnya akan dimulai dari arah sebaliknya: tentukan domain yang punya karakteristik authentic untuk pattern (paralelisme, adversarial, divergent expertise), baru pilih implementasi. Bukan sebaliknya.

## Lampiran: Reference

- `docs/agent-orchestration.md` — arsitektur teknis (apa yang dibangun)
- `docs/orchestrator-prompt-fix.md` — spec prompt fix iterasi terakhir
- `eval/docs/cases-5-7-13-analysis.md` — trace evidence per case
- `eval/docs/session-20260427-handoff.md` — sesi cross-model test
- LangSmith experiment: H-agent_eval (Run A vs Run B comparison)
