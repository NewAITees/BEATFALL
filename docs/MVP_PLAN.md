# BEATFALL MVP Implementation Plan

## 1. Goal and Non-Goals

### Goal
- Build a playable vertical slice where **generated BGM + generated SFX + beat-synced gameplay** feel like one system.
- Keep game loop responsive even when generation is delayed.

### Non-Goals (MVP)
- Real-time streaming generation
- Vocals/lyrics
- Stem separation and score extraction
- Fully automatic loop slicing with robust ML-based segmentation

---

## 2. Recommended Technical Split

- **Game Runtime (main process)**
  - Owns `AudioClock`, `LoopDeck`, `SFXBank`, `EventSequencer`, `MusicState`
  - Never blocks on generation
- **GeneratorService (worker process/thread)**
  - Generates BGM chunks and SFX variants
  - Writes assets to cache + sends availability events

### Why split process/thread
- Prevents frame drops from model inference latency spikes
- Simplifies retry/fallback logic

---

## 3. Directory Layout (Language-Agnostic)

```text
BEATFALL/
  docs/
    MVP_PLAN.md
  game/
    audio/
      audio_clock.*
      loop_deck.*
      sfx_bank.*
      event_sequencer.*
      music_state.*
      mixer_router.*
    gameplay/
      enemy_spawner.*
      wave_director.*
      bullet_pattern.*
      event_bus.*
    config/
      genre_presets.*
      audio_tuning.*
  generator/
    service_main.*
    music_generator.*
    sfx_generator.*
    loop_extractor.*
    cache_store.*
  assets_cache/
    bgm/
    sfx/
```

---

## 4. Core Interfaces (Pseudo)

## 4.1 AudioClock

```text
interface AudioClock {
  bpm_base: float
  bpm_effective: float
  beat_phase: float      // 0..1 in current beat
  bar_index: int

  tick(dt_seconds): void
  quantize(grid): TimePoint   // grid: 1/4, 1/8, 1/16, bar
  set_damage_ratio(ratio): void // affects bpm_effective smoothly
}
```

## 4.2 LoopDeck

```text
interface LoopDeck {
  load_loop(loop_id, audio_clip, bars): void
  play(loop_id, start_time): void
  crossfade_to(loop_id, start_time, fade_seconds=1.5): void
  set_density(density_0_1): void
  set_filter_damage(damage_0_1): void
  set_timestretch_ratio(ratio_0_9_to_1_1): void
}
```

## 4.3 SFXBank

```text
interface SFXBank {
  warmup(category, count): void
  pick(category, seed_hint=None): AudioClip
}
```

Categories:
- `SFX_KILL_A`, `SFX_KILL_B`, `SFX_KILL_C`
- `SFX_DAMAGE`
- `SFX_BOSS_HIT`
- `SFX_BOSS_CLEAR`
- (optional) `SFX_ENV`

## 4.4 EventSequencer

```text
interface EventSequencer {
  enqueue(game_event, quantize_grid, priority): void
  flush_due_events(audio_now): void
}
```

Priority policy:
- `BOSS_CLEAR > DAMAGE > BOSS_HIT > KILL`

## 4.5 GeneratorService

```text
interface GeneratorService {
  request_bgm_chunk(genre, bpm_hint, seconds=30..40): JobId
  request_sfx_pack(category, count=8): JobId
  poll(job_id): JobStatus
  get_result(job_id): GeneratedAsset
}
```

---

## 5. Genre Presets (MVP)

| Genre | BPM Base | Tonal Direction | Notes |
|---|---:|---|---|
| Trance/Techno | 130 | minor, bright top | steady kick, arpeggiated feel |
| DnB | 174 | dark, punchy | half-time perception allowed |
| Lo-fi | 85 | warm, dusty | softer transients |

---

## 6. Runtime State Model

```text
MusicState {
  intensity: 0..1
  density: 0..1
  damage: 0..1
}
```

Update rules (clamped 0..1):
- Kill: `density += 0.02`, `intensity += 0.01`
- Hit: `damage += 0.08`, `density -= 0.08`
- BossClear: `damage *= 0.5`, `intensity += 0.2`

Derived controls:
- `damage -> bpm_effective` (up to -10%)
- `density -> LoopDeck mute/gate amount`
- `damage -> lowpass/bitcrush/glitch send`

---

## 7. Beat-Synced Gameplay Hooks

- Enemy spawn only on beat/bar boundaries:
  - standard mobs: next 1/4 or 1/8
  - elite/wave change: next bar
- Wave phase progression keyed by `bar_index`:
  - e.g. 4 bars = 1 wave segment
- Bullet pattern oscillator keyed by `beat_phase`:
  - pattern transforms every 1/2 or 1 bar

---

## 8. Asset Generation and Cache Strategy

## Startup
1. User selects genre.
2. Request 2 BGM chunks immediately.
3. Extract at least 1 loop (8 or 16 bars).
4. Generate all SFX categories × 8 variants and cache.
5. Enter gameplay only after minimal ready set is available:
   - 1 playable BGM loop
   - `SFX_DAMAGE` + at least one `SFX_KILL_*`

## During play
- Maintain BGM queue depth >= 1 prepared loop.
- If generation late:
  - extend current loop
  - keep gameplay running
  - log generation lag metric

---

## 9. Loop Extraction (MVP Practical)

Semi-automatic approach:
1. Detect rough beat grid from generated chunk.
2. Candidate loop windows: 8/16 bars aligned to downbeat.
3. Score windows by boundary similarity (energy/spectral envelope).
4. Keep top-N and audition quickly (optional editor tool).

Fallback:
- If no good seam, keep full chunk and do bar-aligned re-entry with short crossfade.

---

## 10. Execution Roadmap (2-Week Example)

### Milestone 1: Timing Backbone (Day 1-2)
- Implement `AudioClock`
- Add debug HUD for beat/bar visualization
- Deterministic tests for quantization

### Milestone 2: BGM Loop Runtime (Day 3-4)
- Implement `LoopDeck` with loop playback/crossfade
- Add density + damage mixer controls
- Basic genre preset loader

### Milestone 3: SFX Quantized Events (Day 5-6)
- Implement `SFXBank` and `EventSequencer`
- Route kill/damage/boss events with priorities
- Stress-test simultaneous events

### Milestone 4: Gameplay Sync (Day 7-8)
- Beat-aligned enemy spawn/wave logic
- `beat_phase`-driven bullet variation
- Validate “audio = world law” feel

### Milestone 5: Generation Integration (Day 9-11)
- Implement `GeneratorService` IPC + cache
- Startup warmup pipeline
- In-play prefetch and fallback extension

### Milestone 6: Tuning + Acceptance (Day 12-14)
- Balance `MusicState` deltas and mixer mappings
- 10-minute soak test for stability
- Genre distinguishability check (3 presets)

---

## 11. Acceptance Checklist

- [ ] 10-minute continuous run without audio collapse or dead air
- [ ] Damage state is audible without visuals
- [ ] 3 genres produce clearly different vibe
- [ ] Generation delay never pauses gameplay
- [ ] Event bursts stay musical due to quantize + priority drop

---

## 12. Risks and Mitigations

- **Inference latency spikes**
  - Mitigation: async queue + minimum loop reserve + loop extension fallback
- **Loop seam artifacts**
  - Mitigation: fixed 1-2s crossfade + downbeat-aligned transitions
- **SFX clutter in high-action scenes**
  - Mitigation: priority thinning + per-category cooldown
- **BPM drift vs gameplay timing**
  - Mitigation: single `AudioClock` authority for audio and gameplay

---

## 13. Optional Next Step

If needed, this plan can be expanded into concrete **Python** or **Unity C#** interface files and stubs in the proposed directory structure as a direct starter kit.

---

## 14. heartmula と効果音生成モデルの調査メモ（MVP向け）

> 注記: 本ドキュメント作成時点では、実行環境から外部検索エンジン/公開サイトへの直接アクセス制限があり、
> `heartmula` については一次情報の確認ができていない。以下はMVP実装に必要な観点での整理と、
> 公開情報の確認が取れた時にそのまま差し替えできる評価フレームをまとめたもの。

### 14.1 heartmula（確認観点）

`heartmula` が以下のどれに該当するかで導入方法が変わる。

1. **音楽生成モデル本体**（text-to-music / music continuation）
2. **推論APIサービス名**（内部で複数モデルを切り替える）
3. **ワークフロー/ツール名**（生成 + 後処理を束ねる）

MVP判断で確認すべき最小項目:

- 入力条件: `genre`, `bpm`, `duration`, `seed` を直接指定できるか
- 出力条件: 30〜40秒のステレオWAV/OGGを返せるか
- 利用条件: ローカル実行可否、商用利用可否、レート制限
- 安定性: 同一seedで再現可能か、ノイズ破綻率は許容範囲か

この4点が満たせるなら、MVPの `GeneratorService.request_bgm_chunk()` にそのまま接続可能。

### 14.2 SFX生成モデルの候補（比較）

MVPでは「高忠実度」よりも **短尺を大量生成してバリエーションを確保**できることを優先する。

| 候補タイプ | 向いている用途 | 懸念 | MVP適性 |
|---|---|---|---|
| 汎用テキスト→オーディオ | BossClearや環境ノイズなど長めSFX | 立ち上がりが鈍い場合がある | 中 |
| 効果音特化モデル/サービス | Kill/Damageの短尺ワンショット | サービス依存、利用規約差 | 高 |
| 音楽生成モデルを短尺利用 | ゲーム全体と音色統一しやすい | アタックが甘くなりやすい | 中 |

### 14.3 本MVPでの実運用方針（推奨）

1. **起動時にカテゴリごとに8個生成**（既存仕様通り）
2. 各クリップにメタデータ付与
   - `category`, `duration_ms`, `rms`, `peak`, `spectral_centroid`
3. ゲーム再生前に自動フィルタ
   - クリップ長、ピーク過大、無音率で弾く
4. 採用クリップのみ `SFXBank` に登録

これにより、モデル品質に揺らぎがあってもゲーム内品質を一定に保てる。

### 14.4 追加インターフェース（モデル差し替え可能にする）

```text
interface SFXGeneratorAdapter {
  generate(category, prompt, count, duration_range_ms, seed): List<AudioClip>
  healthcheck(): bool
  model_name(): string
}
```

`GeneratorService` 側でアダプタを差し替え可能にしておくと、
`heartmula`（または代替モデル）検証時にゲーム本体の変更を最小化できる。

### 14.5 調査完了時に更新する項目

- `heartmula` の一次情報リンク（公式ドキュメント/論文/配布先）
- 具体的な推論パラメータ（温度, CFG, steps 相当）
- 生成時間ベンチマーク（GPU種別ごと）
- ライセンス表記テンプレート（ゲーム同梱時）
