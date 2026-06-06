
# VoiceSentinel — Anti-Spoofing Voice Biometric Authenticator

## Complete Project Structure Template

> **Convention:** Every leaf node marked `[FILE]` is a single file.
> Every node without a marker is a directory.
> Inline annotations (after `#`) describe purpose — not code.
> No file in this template contains implementation; this is purely structural.

---

voicesentinel/
│
├── .github/                                    # CI/CD and repository governance
│   ├── workflows/
│   │   ├── ci.yml                              [FILE] # Lint, unit tests, type checks on every PR
│   │   ├── cd.yml                              [FILE] # Build → push Docker images → deploy to staging
│   │   ├── model-eval.yml                      [FILE] # Triggered on model weight updates; runs full benchmark suite
│   │   ├── security-scan.yml                   [FILE] # Trivy container scan + Bandit SAST on every merge to main
│   │   └── proto-generate.yml                  [FILE] # Auto-generates gRPC stubs when .proto files change
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.md                       [FILE]
│   │   ├── feature_request.md                  [FILE]
│   │   └── threat_report.md                    [FILE] # Template for reporting new attack type discoveries
│   └── PULL_REQUEST_TEMPLATE.md                [FILE]
│
│
├── services/                                   # Each subdirectory is an independently deployable microservice
│   │
│   ├── ingestion/                              # Dual-path audio ingestion; WebRTC and SIP/RTP
│   │   │
│   │   ├── webrtc/                             # Path A: Modern / WebRTC-originated calls
│   │   │   ├── livekit/
│   │   │   │   ├── agent.py                    [FILE] # LiveKit Agent entrypoint; registers job handler with LiveKit Cloud/Server
│   │   │   │   ├── audio_stream_handler.py     [FILE] # Subscribes to rtc.AudioStream; emits normalized PCM frames to internal queue
│   │   │   │   ├── session_manager.py          [FILE] # Maps LiveKit room/participant IDs to internal session_id lifecycle
│   │   │   │   ├── participant_tracker.py      [FILE] # Tracks join/leave events; triggers enrollment lookup
│   │   │   │   └── config.py                   [FILE] # LiveKit URL, API key/secret, audio track constraints (16kHz, mono)
│   │   │   └── daily/                          # Alternative WebRTC provider (Daily.co)
│   │   │       ├── client.py                   [FILE] # Daily REST API wrapper for room/token management
│   │   │       └── audio_stream_handler.py     [FILE] # Daily-specific audio frame extraction (mirrors LiveKit handler interface)
│   │   │
│   │   ├── sip_rtp/                            # Path B: Legacy telecom / SIP-trunk-originated calls
│   │   │   ├── siprec_receiver.py              [FILE] # RFC 7866 SIPREC endpoint; receives mirrored RTP from SBC/Genesys/Cisco
│   │   │   ├── rtp_reassembler.py              [FILE] # Reorders out-of-sequence RTP packets; handles SSRC demultiplexing
│   │   │   ├── jitter_buffer.py                [FILE] # Adaptive jitter buffer (target 20ms); handles burst packet loss
│   │   │   ├── codec_decoder.py                [FILE] # Decodes G.711 μ-law, G.722, OPUS to linear PCM @ 16kHz
│   │   │   └── freeswitch/
│   │   │       ├── mod_audio_fork_config.xml   [FILE] # FreeSWITCH mod_audio_fork config; forks RTP to WebSocket without interrupting call
│   │   │       └── dialplan_template.xml       [FILE] # Template dialplan to route calls through anti-spoof middleware
│   │   │
│   │   ├── normalizer.py                       [FILE] # Unified post-ingestion stage: resampling → LUFS normalization → VAD gating → frame chunking
│   │   │                                              # Both WebRTC and SIP/RTP paths converge here; output is identical 160ms PCM frames
│   │   ├── vad_gate.py                         [FILE] # Silero VAD wrapper; suppresses silence frames before they enter the analysis queue
│   │   ├── kafka_producer.py                   [FILE] # Publishes normalized PCM chunks to Kafka topic: audio.{session_id}
│   │   ├── requirements.txt                    [FILE]
│   │   └── Dockerfile                          [FILE]
│   │
│   │
│   ├── analysis/                               # Core anti-spoofing pipeline: DSP features + frozen model inference
│   │   │
│   │   ├── features/                           # Layer 1: Pure DSP feature extraction (zero parameters, ~3ms)
│   │   │   ├── lfcc.py                         [FILE] # Linear Frequency Cepstral Coefficients (60-dim + Δ + ΔΔ = 180-dim)
│   │   │   │                                          # Linear filter bank (not mel) to expose TTS spectral smoothing artifacts
│   │   │   ├── phase_features.py               [FILE] # Group Delay estimation + Instantaneous Frequency deviation
│   │   │   │                                          # Captures phase incoherence signature of neural vocoders (HiFi-GAN, WaveGlow)
│   │   │   ├── subband_energy.py               [FILE] # 8-band energy ratio profile (0–8kHz split)
│   │   │   │                                          # Detects characteristic high-frequency rolloff of replay and VC attacks
│   │   │   ├── modulation_spectrum.py          [FILE] # Amplitude/frequency modulation envelope analysis
│   │   │   │                                          # Synthetic speech is over-smooth in temporal AM; catches diffusion-based TTS
│   │   │   └── pipeline.py                     [FILE] # Orchestrates all DSP extractors; returns unified 214-dim feature vector per frame
│   │   │
│   │   ├── models/                             # Layer 2: Frozen pre-trained models (no training at deployment)
│   │   │   │
│   │   │   ├── aasist_lite/                    # CM model: 297K params, ~4ms CPU inference
│   │   │   │   ├── architecture.py             [FILE] # AASIST-Lite graph definition: sinc-filter front-end → 3 conv blocks → HS-GAL → readout
│   │   │   │   ├── graph_attention.py          [FILE] # Heterogeneous Spectro-temporal Graph Attention Layer (HS-GAL) definition
│   │   │   │   ├── sinc_filters.py             [FILE] # Learnable sinc filter bank (fixed at inference; pre-trained values loaded from checkpoint)
│   │   │   │   ├── loader.py                   [FILE] # Loads .pt checkpoint → sets model.eval() → freezes all parameters
│   │   │   │   └── inference.py                [FILE] # Single-sample and batched inference interface; returns spoof_score ∈ [0,1]
│   │   │   │
│   │   │   └── ecapa_tdnn/                     # ASV model: 5.7M params (SpeechBrain spkrec-ecapa-voxceleb), ~4ms CPU
│   │   │       ├── loader.py                   [FILE] # Loads SpeechBrain pre-trained checkpoint; freezes all parameters
│   │   │       └── inference.py                [FILE] # Encodes audio → 192-dim speaker embedding; used for cosine similarity ASV scoring
│   │   │
│   │   ├── ensemble/                           # Dual-checkpoint ensemble for adversarial robustness (no extra training)
│   │   │   ├── manager.py                      [FILE] # Loads two AASIST-Lite checkpoints (ASVspoof19-LA weights + ASVspoof21-DF weights)
│   │   │   └── disagreement_detector.py        [FILE] # Flags sessions where ensemble score std-dev exceeds threshold → adversarial suspect
│   │   │
│   │   ├── sprt.py                             [FILE] # Sequential Probability Ratio Test engine
│   │   │                                              # Processes 40ms audio chunks; makes early stopping decision when evidence crosses A or B boundary
│   │   │                                              # Avoids waiting for full 160ms window for high-confidence cases
│   │   ├── adversarial_defense.py              [FILE] # Randomized smoothing inference wrapper (adds Gaussian noise n=50 samples, σ=0.005)
│   │   │                                              # Routes ambiguous/flagged sessions through diffusion-based purification
│   │   ├── kafka_consumer.py                   [FILE] # Consumes from audio.{session_id} Kafka topic; dispatches frames to analysis pipeline
│   │   ├── kafka_producer.py                   [FILE] # Publishes raw scores to scores.{session_id} Kafka topic
│   │   ├── requirements.txt                    [FILE]
│   │   └── Dockerfile                          [FILE]
│   │
│   │
│   ├── active_liveness/                        # Challenge-response liveness service
│   │   │
│   │   ├── challenge/
│   │   │   ├── generator.py                    [FILE] # Generates session-unique phoneme/digit challenge strings from high-entropy pool
│   │   │   │                                          # Ensures no challenge reuse within 24-hour window per speaker_id
│   │   │   ├── phoneme_pool.py                 [FILE] # Curated phoneme sequences designed for maximum coarticulation variability
│   │   │   │                                          # Includes multilingual variants (English, Hindi, Mandarin, Spanish, Arabic)
│   │   │   └── session_tracker.py              [FILE] # Tracks challenge issuance, expiry (30s TTL), and response state per session
│   │   │
│   │   ├── response_validator.py               [FILE] # ASR-validates that the spoken response matches the issued challenge text
│   │   ├── latency_oracle.py                   [FILE] # Measures onset latency (challenge delivery → first phoneme energy detected)
│   │   │                                              # Flags latency outside human range [180ms, 450ms] as pipeline indicator
│   │   ├── coarticulation_scorer.py            [FILE] # Scores phoneme transition naturalness against challenge-specific acoustic model
│   │   │                                              # Detects absence of genuine coarticulation in pre-synthesized or spliced responses
│   │   ├── kafka_consumer.py                   [FILE] # Consumes audio frames during challenge window
│   │   ├── kafka_producer.py                   [FILE] # Publishes active_liveness_score to decisions.{session_id} topic
│   │   ├── requirements.txt                    [FILE]
│   │   └── Dockerfile                          [FILE]
│   │
│   │
│   ├── decision/                               # Score fusion, policy engine, and explainability
│   │   │
│   │   ├── fusion/
│   │   │   ├── score_fusion.py                 [FILE] # Combines spoof_score (CM), cosine_sim (ASV), active_liveness_score into joint posterior
│   │   │   │                                          # Implements SASV joint scoring: S_joint = λ·S_ASV + (1-λ)·S_CM with learned λ
│   │   │   ├── bayesian_posterior.py           [FILE] # Calibrated Bayesian fusion of passive + active + latency scores
│   │   │   └── calibration.py                  [FILE] # Platt scaling / isotonic regression calibration for raw model scores
│   │   │                                              # Ensures spoof_score outputs are true probabilities, not uncalibrated logits
│   │   │
│   │   ├── policy/
│   │   │   ├── engine.py                       [FILE] # Applies customer-configured risk policy to joint_score → AuthDecision
│   │   │   ├── risk_tiers.py                   [FILE] # Defines LOW / MEDIUM / HIGH / CRITICAL risk tier boundaries
│   │   │   └── thresholds.py                   [FILE] # Per-tenant configurable decision thresholds (loaded from config store)
│   │   │                                              # Supports asymmetric cost weighting: FAS penalty >> FRR penalty
│   │   │
│   │   ├── explainability/
│   │   │   ├── evidence_builder.py             [FILE] # Assembles ArtifactEvidence payload from per-feature percentile scores
│   │   │   │                                          # Produces human-readable, compliance-defensible explanation per decision
│   │   │   └── attack_type_classifier.py       [FILE] # Heuristic classifier over feature signatures → TTS | VC | REPLAY | ADVERSARIAL | UNKNOWN
│   │   │
│   │   ├── ood_detector.py                     [FILE] # Energy-based OOD scoring: E(x;f) = -T·log Σ_c exp(f_c(x)/T)
│   │   │                                              # Flags genuinely novel / unseen attack types for human review queue
│   │   ├── kafka_consumer.py                   [FILE] # Consumes scores.{session_id} from analysis service
│   │   ├── redis_writer.py                     [FILE] # Writes final AuthDecision to Redis (TTL 300s) for synchronous API polling
│   │   ├── webhook_dispatcher.py               [FILE] # Pushes AuthDecision to customer-configured webhook URL on decision completion
│   │   ├── requirements.txt                    [FILE]
│   │   └── Dockerfile                          [FILE]
│   │
│   │
│   ├── enrollment/                             # Speaker profile creation and management
│   │   ├── processor.py                        [FILE] # Accepts 3–5 enrollment audio samples; validates quality; extracts ECAPA embeddings
│   │   ├── quality_checker.py                  [FILE] # Rejects enrollment audio below SNR threshold, too short, or VAD-detected non-speech
│   │   ├── embedding_aggregator.py             [FILE] # Averages N enrollment embeddings → single 192-dim speaker model vector
│   │   ├── embedding_store.py                  [FILE] # Redis interface for storing/retrieving speaker embeddings keyed by speaker_id
│   │   ├── anti_spoof_guard.py                 [FILE] # Runs CM on enrollment audio; rejects enrollment if spoof_score > 0.3
│   │   │                                              # Prevents poisoning of speaker model with synthetic enrollment audio
│   │   ├── requirements.txt                    [FILE]
│   │   └── Dockerfile                          [FILE]
│   │
│   │
│   ├── api_gateway/                            # External-facing API: gRPC (primary) + REST (compatibility)
│   │   │
│   │   ├── grpc/
│   │   │   ├── server.py                       [FILE] # gRPC server bootstrap; registers all service implementations; configures TLS
│   │   │   ├── interceptors/
│   │   │   │   ├── auth_interceptor.py         [FILE] # Validates API key or mTLS client cert per tenant; attaches tenant context
│   │   │   │   ├── rate_limiter.py             [FILE] # Per-tenant + per-session rate limiting; exponential backoff on auth failures
│   │   │   │   ├── request_logger.py           [FILE] # Structured logging of all gRPC calls (metadata only; no audio content)
│   │   │   │   └── tracing_interceptor.py      [FILE] # OpenTelemetry span injection for distributed tracing
│   │   │   └── handlers/
│   │   │       ├── stream_handler.py           [FILE] # Implements StreamAudio RPC: accepts audio stream → pushes to Kafka → polls Redis for decision
│   │   │       ├── enrollment_handler.py       [FILE] # Implements EnrollSpeaker RPC: validates → processes → stores speaker model
│   │   │       ├── decision_handler.py         [FILE] # Implements GetDecision RPC: synchronous pull from Redis decision store
│   │   │       └── fraud_feedback_handler.py   [FILE] # Implements ReportFraud RPC: ingests human-review outcomes for threat intel feed
│   │   │
│   │   ├── rest/                               # REST layer for customers not using gRPC
│   │   │   ├── app.py                          [FILE] # FastAPI application factory; mounts all routers; configures CORS and middleware
│   │   │   ├── routers/
│   │   │   │   ├── sessions.py                 [FILE] # POST /v1/sessions, GET /v1/sessions/{id}/decision
│   │   │   │   ├── enrollment.py               [FILE] # POST /v1/speakers, DELETE /v1/speakers/{id}
│   │   │   │   ├── forensics.py                [FILE] # POST /v1/forensics/analyze (batch, async)
│   │   │   │   └── health.py                   [FILE] # GET /health, GET /ready — liveness and readiness probes
│   │   │   └── middleware/
│   │   │       ├── auth.py                     [FILE] # Bearer token / API key validation
│   │   │       └── request_id.py               [FILE] # Injects X-Request-ID header for trace correlation
│   │   │
│   │   ├── websocket/
│   │   │   └── audio_bridge.py                 [FILE] # WebSocket endpoint for customers streaming raw audio directly to the gateway
│   │   │                                              # Bridges browser/app WebSocket → internal Kafka producer
│   │   │
│   │   ├── openapi.yaml                        [FILE] # OpenAPI 3.1 spec (auto-generated from FastAPI; source of truth for REST docs)
│   │   ├── requirements.txt                    [FILE]
│   │   └── Dockerfile                          [FILE]
│   │
│   │
│   ├── threat_intel/                           # Threat intelligence network and model update service
│   │   │
│   │   ├── feed/
│   │   │   ├── collector.py                    [FILE] # Aggregates anonymized artifact signatures from fraud_feedback across tenants
│   │   │   │                                          # Strips all PII/audio content; retains only feature-space representations
│   │   │   ├── aggregator.py                   [FILE] # Clusters collected signatures; identifies novel attack type patterns
│   │   │   └── publisher.py                    [FILE] # Publishes threat intel updates to subscriber tenants via signed update channel
│   │   │
│   │   ├── model_updater/
│   │   │   ├── checkpoint_manager.py           [FILE] # Manages versioned model weight updates; validates signature before deployment
│   │   │   ├── canary_deployer.py              [FILE] # Rolls updated weights to 5% of traffic first; compares EER before full rollout
│   │   │   └── rollback.py                     [FILE] # Reverts to previous checkpoint if EER degrades > 1% absolute after update
│   │   │
│   │   ├── signature_db/
│   │   │   ├── schema.sql                      [FILE] # PostgreSQL schema for attack signature storage (feature vectors, attack_type, confidence, timestamp)
│   │   │   └── migrations/                     # Alembic migration files
│   │   │       └── .gitkeep
│   │   │
│   │   ├── requirements.txt                    [FILE]
│   │   └── Dockerfile                          [FILE]
│   │
│   │
│   └── forensics/                              # Batch / post-hoc analysis for fraud investigation and compliance
│       ├── batch_analyzer.py                   [FILE] # Accepts list of audio file paths or S3 URIs; runs full pipeline asynchronously
│       ├── report_generator.py                 [FILE] # Produces structured JSON + PDF forensic report per analyzed session
│       │                                              # Report includes: per-feature evidence, confidence intervals, attack classification
│       ├── timeline_reconstructor.py           [FILE] # Reconstructs call timeline from session logs; maps auth decisions to call events
│       ├── export_formats/
│       │   ├── json_exporter.py                [FILE] # Machine-readable export for SIEM integration
│       │   └── pdf_exporter.py                 [FILE] # Human-readable PDF for legal/compliance teams
│       ├── requirements.txt                    [FILE]
│       └── Dockerfile                          [FILE]
│
│
├── models/                                     # Model weights, configs, and ONNX exports (git-lfs tracked)
│   │
│   ├── weights/                                # Pre-trained checkpoint files (never committed to git; managed via DVC or git-lfs)
│   │   ├── aasist_lite/
│   │   │   ├── asvspoof19_la.pt                [FILE] # Official AASIST-Lite weights trained on ASVspoof 2019 LA partition
│   │   │   ├── asvspoof21_df.pt                [FILE] # Official AASIST-Lite weights trained on ASVspoof 2021 DF partition
│   │   │   └── .gitkeep                        [FILE]
│   │   └── ecapa_tdnn/
│   │       ├── voxceleb2_cs.pt                 [FILE] # SpeechBrain spkrec-ecapa-voxceleb-cs checkpoint
│   │       └── .gitkeep                        [FILE]
│   │
│   ├── onnx/                                   # TensorRT-optimized ONNX exports (generated by scripts/export_onnx.py)
│   │   ├── aasist_lite_int8.onnx               [FILE] # INT8 quantized; ~4x faster than FP32 baseline
│   │   ├── ecapa_tdnn_fp16.onnx                [FILE] # FP16; minimal accuracy loss vs FP32
│   │   └── .gitkeep                            [FILE]
│   │
│   ├── configs/                                # Model architecture hyperparameters (referenced by loader.py in each service)
│   │   ├── aasist_lite.yaml                    [FILE] # n_sinc_filters, n_conv_blocks, graph_heads, embedding_dim, etc.
│   │   └── ecapa_tdnn.yaml                     [FILE] # channels, kernel_sizes, dilation, attention_dim, embedding_dim
│   │
│   ├── registry/
│   │   └── model_registry.yaml                 [FILE] # Maps model_name → checkpoint_path, version, eval_results, deployment_status
│   │                                                  # Single source of truth for which checkpoint is active in each environment
│   │
│   └── dvc.yaml                                [FILE] # DVC pipeline for reproducible model weight downloads and version tracking
│
│
├── proto/                                      # gRPC Protocol Buffer definitions
│   ├── anti_spoof.proto                        [FILE] # AntiSpoofService: CreateSession, StreamAudio, GetDecision RPCs
│   │                                                  # AuthDecision message: spoof_score, asv_score, joint_score, risk_tier, attack_type, evidence[]
│   ├── enrollment.proto                        [FILE] # EnrollmentService: EnrollSpeaker, DeleteSpeaker, GetSpeakerModel RPCs
│   ├── forensics.proto                         [FILE] # ForensicsService: SubmitBatch, GetBatchStatus, GetReport RPCs
│   ├── common.proto                            [FILE] # Shared message types: ArtifactEvidence, RiskTier enum, AttackHypothesis enum
│   └── generated/                              # Auto-generated stubs (do not edit manually; regenerated by proto-generate.yml CI workflow)
│       ├── python/
│       │   └── .gitkeep
│       ├── go/
│       │   └── .gitkeep
│       └── node/
│           └── .gitkeep
│
│
├── sdk/                                        # Client SDKs for enterprise integrations
│   │
│   ├── python/                                 # Pip-installable: pip install voicesentinel
│   │   ├── voicesentinel/
│   │   │   ├── __init__.py                     [FILE]
│   │   │   ├── client.py                       [FILE] # VoiceSentinelClient: wraps gRPC stubs with auth, retry, and timeout logic
│   │   │   ├── stream.py                       [FILE] # AudioStream helper: accepts file path, bytes, or generator → streams to API
│   │   │   ├── models.py                       [FILE] # Pydantic models mirroring proto messages (AuthDecision, EnrollmentResult, etc.)
│   │   │   └── exceptions.py                   [FILE] # SpoofDetectedError, SessionExpiredError, EnrollmentRejectedError, etc.
│   │   ├── examples/
│   │   │   ├── basic_verification.py           [FILE] # Minimal working example: enroll → stream → get decision
│   │   │   ├── livekit_integration.py          [FILE] # Full LiveKit agent with VoiceSentinel integrated
│   │   │   └── twilio_integration.py           [FILE] # Twilio Media Streams → VoiceSentinel pipeline
│   │   ├── pyproject.toml                      [FILE]
│   │   └── README.md                           [FILE]
│   │
│   ├── node/                                   # npm package: voicesentinel-sdk
│   │   ├── src/
│   │   │   ├── client.ts                       [FILE] # TypeScript gRPC client wrapper
│   │   │   ├── stream.ts                       [FILE] # Readable stream adapter for browser MediaStream → API
│   │   │   └── types.ts                        [FILE] # TypeScript type definitions mirroring proto messages
│   │   ├── examples/
│   │   │   ├── vapi_integration.ts             [FILE] # Vapi.ai webhook → VoiceSentinel verification flow
│   │   │   └── retell_integration.ts           [FILE] # Retell.ai custom function → VoiceSentinel flow
│   │   ├── package.json                        [FILE]
│   │   └── README.md                           [FILE]
│   │
│   └── go/                                     # Go module: github.com/voicesentinel/sdk-go
│       ├── client.go                           [FILE] # Go gRPC client with context propagation and retry middleware
│       ├── stream.go                           [FILE] # io.Reader adapter for streaming audio
│       ├── go.mod                              [FILE]
│       └── README.md                           [FILE]
│
│
├── datasets/                                   # Dataset management, augmentation, and loader utilities
│   │
│   ├── management/
│   │   ├── registry.yaml                       [FILE] # Catalog of all datasets: name, source URL, license, split sizes, download instructions
│   │   │                                              # Covers: ASVspoof5, ASVspoof2021-DF, WaveFake, In-the-Wild, MLAAD, HABLA, VoxCeleb2
│   │   ├── downloader.py                       [FILE] # Automated download + integrity check (MD5/SHA256) for each registered dataset
│   │   └── splitter.py                         [FILE] # Creates deterministic train/val/test splits; ensures no speaker overlap across splits
│   │
│   ├── augmentation/                           # Real-world degradation simulation pipeline (applied during offline feature prep)
│   │   ├── codec_chain.py                      [FILE] # ffmpeg-based codec round-trip simulation: G.711, G.729, AMR-NB, OPUS, G.722
│   │   │                                              # Applies encode → decode → upsample to 16kHz; preserves codec-specific artifact signatures
│   │   ├── packet_loss.py                      [FILE] # Gilbert-Elliott two-state Markov model for burst packet loss simulation
│   │   │                                              # Applies Packet Loss Concealment (PLC) on lost frames as real RTP stacks do
│   │   ├── rir_convolver.py                    [FILE] # Convolves audio with measured Room Impulse Responses (OpenSLR26/28 RIR databases)
│   │   │                                              # Simulates near-field, far-field, reverberant environments
│   │   ├── noise_mixer.py                      [FILE] # Mixes MUSAN noise (music, speech, noise classes) at target SNR ∈ [-5dB, 25dB]
│   │   ├── microphone_sim.py                   [FILE] # Applies measured microphone frequency response profiles (Bluetooth, laptop mic, headset)
│   │   ├── lufs_normalizer.py                  [FILE] # ITU-R BS.1770 LUFS normalization to -23 LUFS with ±3 LUFS random variation
│   │   └── pipeline.py                         [FILE] # Stochastic augmentation orchestrator: applies subset of augmentations per sample
│   │                                                  # Enforces correct ordering: RIR → noise → codec → packet_loss (not arbitrary)
│   │
│   └── loaders/                                # PyTorch Dataset classes for each corpus
│       ├── asvspoof5.py                        [FILE] # ASVspoof 5 (2024) loader; handles open-condition audio with metadata parsing
│       ├── asvspoof21_df.py                    [FILE] # ASVspoof 2021 DF track; codec-compressed synthetic audio
│       ├── wavefake.py                         [FILE] # WaveFake; 7 vocoder architectures; returns vocoder_type label alongside binary label
│       ├── in_the_wild.py                      [FILE] # In-the-Wild dataset (Yi et al., 2022); real internet-sourced deepfakes
│       ├── mlaad.py                            [FILE] # Multi-Language Audio Anti-Spoofing Dataset; 23 languages, 54 TTS systems
│       └── voxceleb2.py                        [FILE] # VoxCeleb2 loader for ASV speaker encoder pre-training (not used at inference)
│
│
├── evaluation/                                 # Benchmarking, metrics, and adversarial robustness testing
│   │
│   ├── metrics/
│   │   ├── eer.py                              [FILE] # Equal Error Rate computation; returns EER and decision threshold at EER operating point
│   │   ├── sasv_eer.py                         [FILE] # SASV-EER: joint metric over {genuine, non-target, spoof} three-class scenario
│   │   ├── tdcf.py                             [FILE] # Tandem Detection Cost Function (t-DCF); normalized cost weighting FAS and FRR asymmetrically
│   │   ├── latency_profiler.py                 [FILE] # End-to-end latency measurement: ingestion → feature → model → decision; P50/P95/P99
│   │   └── calibration_metrics.py              [FILE] # Expected Calibration Error (ECE); reliability diagrams for score calibration assessment
│   │
│   ├── benchmarks/                             # Reproducible benchmark scripts against each evaluation corpus
│   │   ├── run_asvspoof19_la.py                [FILE] # Evaluates full pipeline on ASVspoof 2019 LA eval set; reports EER + t-DCF
│   │   ├── run_asvspoof21_df.py                [FILE] # Evaluates on ASVspoof 2021 DF (codec-compressed conditions)
│   │   ├── run_in_the_wild.py                  [FILE] # Held-out real-world evaluation; most predictive of deployed performance
│   │   ├── run_mlaad.py                        [FILE] # Multilingual evaluation across all 23 language subsets
│   │   └── run_latency.py                      [FILE] # Measures inference latency under simulated concurrent load (10, 50, 100 streams)
│   │
│   ├── adversarial/                            # Robustness evaluation against adversarial audio attacks
│   │   ├── fgsm_attack.py                      [FILE] # Fast Gradient Sign Method attack on CM model; measures robustness degradation
│   │   ├── pgd_attack.py                       [FILE] # Projected Gradient Descent attack; stronger iterative attack baseline
│   │   ├── transfer_attack.py                  [FILE] # Crafts adversarial examples on surrogate model; evaluates black-box transferability
│   │   └── certified_robustness_eval.py        [FILE] # Measures certified L2 robustness radius under randomized smoothing defense
│   │
│   └── reports/                                # Auto-generated benchmark output artifacts
│       └── .gitkeep                            [FILE] # Populated by CI model-eval.yml workflow; not committed to git
│
│
├── infra/                                      # All deployment and infrastructure configuration
│   │
│   ├── docker/
│   │   ├── docker-compose.yml                  [FILE] # Full local stack: all services + Kafka + Redis + PostgreSQL + Prometheus
│   │   ├── docker-compose.dev.yml              [FILE] # Lightweight dev override: single-service mode with mocked Kafka
│   │   └── docker-compose.prod.yml             [FILE] # Production-hardened: no exposed ports, TLS everywhere, read-only filesystems
│   │
│   ├── kubernetes/
│   │   ├── namespaces/
│   │   │   └── voicesentinel.yaml              [FILE] # Namespace definition with ResourceQuota and LimitRange
│   │   ├── deployments/
│   │   │   ├── ingestion.yaml                  [FILE] # Deployment spec: replicas, resource limits, liveness/readiness probes
│   │   │   ├── analysis.yaml                   [FILE] # GPU node selector (nvidia.com/gpu: 1), tolerations, model weight volume mount
│   │   │   ├── active_liveness.yaml            [FILE]
│   │   │   ├── decision.yaml                   [FILE]
│   │   │   ├── enrollment.yaml                 [FILE]
│   │   │   ├── api_gateway.yaml                [FILE] # Service type: LoadBalancer; TLS termination via cert-manager
│   │   │   ├── threat_intel.yaml               [FILE]
│   │   │   └── forensics.yaml                  [FILE]
│   │   ├── services/
│   │   │   └── *.yaml                          [FILE] # ClusterIP services for internal inter-service communication
│   │   ├── configmaps/
│   │   │   ├── analysis-config.yaml            [FILE] # Model paths, SPRT thresholds, ensemble config
│   │   │   └── decision-config.yaml            [FILE] # Risk tier boundaries, default decision thresholds
│   │   ├── secrets/
│   │   │   └── .gitkeep                        [FILE] # External Secrets Operator or Sealed Secrets; never plaintext secrets in git
│   │   ├── hpa/
│   │   │   ├── analysis-hpa.yaml               [FILE] # HorizontalPodAutoscaler: scales analysis pods on GPU utilization metric
│   │   │   └── ingestion-hpa.yaml              [FILE] # Scales ingestion pods on Kafka consumer lag
│   │   ├── keda/
│   │   │   └── kafka-scaler.yaml               [FILE] # KEDA ScaledObject: event-driven autoscaling on Kafka topic lag
│   │   ├── network_policies/
│   │   │   └── deny-egress.yaml                [FILE] # Default-deny egress; allowlist only required inter-service and external endpoints
│   │   └── pdb/
│   │       └── analysis-pdb.yaml               [FILE] # PodDisruptionBudget: minimum 2 analysis replicas always available during rollouts
│   │
│   ├── helm/
│   │   └── voicesentinel/
│   │       ├── Chart.yaml                      [FILE] # Chart metadata: name, version, appVersion, description, dependencies
│   │       ├── values.yaml                     [FILE] # Default values: replica counts, image tags, resource limits, feature flags
│   │       ├── values.staging.yaml             [FILE] # Staging overrides: reduced replicas, debug logging enabled
│   │       ├── values.prod.yaml                [FILE] # Production overrides: full replicas, PDB enabled, stricter thresholds
│   │       ├── values.edge.yaml                [FILE] # On-premise / air-gapped: no cloud dependencies, local model weights, CPU-only inference
│   │       └── templates/                      # Helm template files (one per Kubernetes resource type)
│   │           └── .gitkeep
│   │
│   ├── terraform/
│   │   ├── modules/
│   │   │   ├── aws/
│   │   │   │   ├── eks_cluster.tf              [FILE] # EKS cluster with GPU node group (g4dn.xlarge)
│   │   │   │   ├── kafka_msk.tf                [FILE] # Amazon MSK (managed Kafka) cluster configuration
│   │   │   │   ├── elasticache_redis.tf        [FILE] # ElastiCache Redis for speaker embeddings and decision cache
│   │   │   │   └── s3_audit_bucket.tf          [FILE] # Encrypted S3 bucket for flagged audio retention (90-day lifecycle)
│   │   │   ├── gcp/
│   │   │   │   ├── gke_cluster.tf              [FILE] # GKE Autopilot with T4 GPU node pool
│   │   │   │   └── pubsub.tf                   [FILE] # Cloud Pub/Sub as Kafka alternative for GCP deployments
│   │   │   └── azure/
│   │   │       └── aks_cluster.tf              [FILE] # AKS with NC4as_T4_v3 GPU node pool
│   │   ├── environments/
│   │   │   ├── dev/
│   │   │   │   └── main.tf                     [FILE]
│   │   │   ├── staging/
│   │   │   │   └── main.tf                     [FILE]
│   │   │   └── prod/
│   │   │       └── main.tf                     [FILE]
│   │   └── variables.tf                        [FILE] # Input variables shared across all modules
│   │
│   └── triton/                                 # NVIDIA Triton Inference Server configuration
│       ├── model_repository/
│       │   ├── aasist_lite_int8/
│       │   │   ├── config.pbtxt                [FILE] # Triton model config: backend=onnxruntime, max_batch_size=32, dynamic_batching{max_queue_delay=20ms}
│       │   │   └── 1/
│       │   │       └── .gitkeep                [FILE] # ONNX model file placed here at deployment (not in git)
│       │   └── ecapa_tdnn_fp16/
│       │       ├── config.pbtxt                [FILE] # Triton config for speaker encoder: input shape, output shape, precision
│       │       └── 1/
│       │           └── .gitkeep                [FILE]
│       └── perf_analyzer_config.yaml           [FILE] # Triton perf_analyzer benchmark config: concurrency sweep 1→100 streams
│
│
├── config/                                     # Environment-specific configuration (no secrets; secrets managed externally)
│   ├── base.yaml                               [FILE] # Shared defaults: Kafka bootstrap servers, Redis URL, model registry path,
│   │                                                  # SPRT thresholds (A, B), decision timeout, VAD aggressiveness level
│   ├── development.yaml                        [FILE] # Local overrides: mock Kafka, in-memory Redis, local model weight paths
│   ├── staging.yaml                            [FILE] # Staging cluster config: relaxed thresholds, verbose logging, test tenant IDs
│   ├── production.yaml                         [FILE] # Production: strict thresholds, audit logging, PII scrubbing enabled
│   └── edge.yaml                               [FILE] # On-premise deployment: all endpoints localhost, no external calls, air-gap safe
│
│
├── tests/
│   │
│   ├── unit/                                   # Fast, isolated tests with no external dependencies
│   │   ├── features/
│   │   │   ├── test_lfcc.py                    [FILE] # LFCC output shape, numerical stability, edge cases (silence, clipping)
│   │   │   ├── test_phase_features.py          [FILE] # Group delay and IF computation correctness against reference implementation
│   │   │   ├── test_subband_energy.py          [FILE] # Energy ratio normalization; sum-to-one property
│   │   │   └── test_modulation_spectrum.py     [FILE]
│   │   ├── models/
│   │   │   ├── test_aasist_lite_loader.py      [FILE] # Verifies checkpoint loads correctly; all parameters frozen; output shape correct
│   │   │   └── test_ecapa_tdnn_loader.py       [FILE] # Verifies embedding dimensionality (192); frozen param check
│   │   ├── decision/
│   │   │   ├── test_score_fusion.py            [FILE] # Joint posterior computation; edge cases (both scores 0, both 1)
│   │   │   ├── test_policy_engine.py           [FILE] # Risk tier assignment for all threshold boundary conditions
│   │   │   ├── test_ood_detector.py            [FILE] # Energy score ordering: in-distribution < out-of-distribution
│   │   │   └── test_evidence_builder.py        [FILE] # Evidence payload structure and completeness
│   │   ├── active_liveness/
│   │   │   ├── test_challenge_generator.py     [FILE] # Uniqueness guarantee within 24h window; entropy validation
│   │   │   └── test_latency_oracle.py          [FILE] # Boundary conditions: exactly 180ms, exactly 450ms, 179ms (fail), 451ms (fail)
│   │   └── ingestion/
│   │       └── test_normalizer.py              [FILE] # LUFS normalization accuracy; resampling correctness; frame size output
│   │
│   ├── integration/                            # Tests spanning two or more services; use docker-compose.dev.yml
│   │   ├── test_ingestion_to_analysis.py       [FILE] # Audio frame flows from WebSocket → Kafka → analysis consumer correctly
│   │   ├── test_analysis_to_decision.py        [FILE] # Score published to Kafka → decision service produces AuthDecision in Redis
│   │   ├── test_enrollment_pipeline.py         [FILE] # Full enrollment: upload audio → quality check → embedding → Redis storage
│   │   └── test_end_to_end.py                  [FILE] # Full pipeline: enroll speaker → stream audio → poll GetDecision → verify response shape
│   │
│   ├── load/                                   # Performance and concurrency tests
│   │   ├── k6/
│   │   │   ├── streaming_load.js               [FILE] # k6 script: 100 concurrent WebSocket audio streams; validates P99 latency < 500ms
│   │   │   └── enrollment_load.js              [FILE] # k6 script: burst enrollment requests; validates no embedding collision
│   │   └── locust/
│   │       └── rest_api_load.py                [FILE] # Locust scenario: mixed enroll + stream + decision traffic pattern
│   │
│   ├── adversarial/
│   │   ├── test_fgsm_robustness.py             [FILE] # Verifies ensemble disagreement detection catches FGSM-perturbed audio
│   │   └── test_randomized_smoothing.py        [FILE] # Verifies certified robustness radius meets minimum requirement (r > 0.003)
│   │
│   └── fixtures/
│       ├── audio/
│       │   ├── genuine/                        # Short clean speech samples for testing (< 5s; synthetic speakers; no real PII)
│       │   ├── replay/                         # Simulated replay artifacts at various quality levels
│       │   ├── tts/                            # Samples from public TTS systems (ElevenLabs free tier, gTTS, etc.)
│       │   └── vc/                             # Voice-converted samples using open-source VC systems
│       └── speaker_profiles/
│           └── .gitkeep                        # Pre-computed ECAPA embeddings for test speakers (numpy .npy files)
│
│
├── research/                                   # Exploratory work; not part of production build
│   │
│   ├── notebooks/
│   │   ├── 01_feature_analysis.ipynb           [FILE] # Visualizes LFCC, phase, sub-band features across attack types; t-SNE plots
│   │   ├── 02_aasist_lite_benchmark.ipynb      [FILE] # Reproduces published AASIST-Lite EER numbers; validates local checkpoint
│   │   ├── 03_codec_augmentation_study.ipynb   [FILE] # Ablation: EER vs. codec augmentation intensity; finds optimal augmentation rate
│   │   ├── 04_latency_profiling.ipynb          [FILE] # End-to-end latency breakdown; identifies bottleneck (feature vs. model vs. network)
│   │   ├── 05_adversarial_attack_study.ipynb   [FILE] # Visualizes FGSM/PGD perturbation in spectrogram space; measures perceptibility
│   │   └── 06_ensemble_disagreement.ipynb      [FILE] # Analyzes ensemble score distributions; calibrates disagreement threshold
│   │
│   ├── experiments/
│   │   └── .gitkeep                            # Tracked experiment artifacts (MLflow or W&B run exports)
│   │
│   └── papers/
│       └── references.bib                      [FILE] # BibTeX references: AASIST, W2V-AASIST, SASV challenge, SPRT, ECAPA-TDNN, WavLM, etc.
│
│
├── monitoring/                                 # Observability stack configuration
│   │
│   ├── prometheus/
│   │   ├── rules/
│   │   │   ├── latency_alerts.yaml             [FILE] # Alert: P99 latency > 450ms sustained for > 60s
│   │   │   ├── accuracy_drift_alerts.yaml      [FILE] # Alert: rolling spoof_score mean shifts > 0.15 from baseline (model drift indicator)
│   │   │   ├── fraud_spike_alerts.yaml         [FILE] # Alert: spoof detection rate > 20% in any 5-min window (active attack indicator)
│   │   │   └── ensemble_disagreement_alerts.yaml [FILE] # Alert: disagreement rate > 5% (possible adversarial campaign)
│   │   └── scrape_config.yaml                  [FILE] # Prometheus scrape targets for all services
│   │
│   ├── grafana/
│   │   └── dashboards/
│   │       ├── system_health.json              [FILE] # CPU, GPU utilization, memory, Kafka lag, Redis hit rate
│   │       ├── spoof_detection_metrics.json    [FILE] # Real-time: spoof_rate, genuine_rate, risk_tier distribution, attack_type breakdown
│   │       └── latency_breakdown.json          [FILE] # Per-stage latency: feature extraction, CM inference, ASV inference, fusion, API response
│   │
│   └── opentelemetry/
│       └── collector_config.yaml               [FILE] # OTel Collector: receives traces from all services; exports to Jaeger or OTLP endpoint
│
│
├── scripts/                                    # Developer and operator utility scripts
│   ├── setup_dev.sh                            [FILE] # One-command local dev setup: installs deps, downloads model weights, starts docker-compose
│   ├── download_models.sh                      [FILE] # Downloads AASIST-Lite and ECAPA-TDNN checkpoints; validates checksums
│   ├── download_datasets.sh                    [FILE] # Invokes datasets/management/downloader.py for all registered datasets
│   ├── export_onnx.sh                          [FILE] # Exports all .pt checkpoints to ONNX with INT8/FP16 quantization; places in models/onnx/
│   ├── generate_proto.sh                       [FILE] # Runs protoc for all .proto files; outputs stubs to proto/generated/{python,go,node}
│   ├── benchmark.sh                            [FILE] # Runs full evaluation suite against all benchmark corpora; outputs report to evaluation/reports/
│   ├── run_load_test.sh                        [FILE] # Starts k6 load test against local or staging environment
│   └── rotate_model.sh                         [FILE] # Operator script: validates new checkpoint → updates model_registry.yaml → triggers canary deploy
│
│
├── .env.example                                [FILE] # Template: all required env vars with descriptions; no actual values
├── .gitignore                                  [FILE] # Excludes: model weights (*.pt, *.onnx), dataset audio files, .env, ___pycache__, reports/
├── .dockerignore                               [FILE] # Excludes: research/, tests/, datasets/, .github/ from Docker build context
├── .pre-commit-config.yaml                     [FILE] # Pre-commit hooks: ruff lint, mypy type check, hadolint Dockerfile lint, detect-secrets
├── pyproject.toml                              [FILE] # Root Python project config: ruff, mypy, pytest settings; workspace dependencies
├── Makefile                                    [FILE] # Convenience targets: make dev, make test, make benchmark, make deploy-staging, make proto
├── SECURITY.md                                 [FILE] # Responsible disclosure policy; instructions for reporting vulnerabilities and new attack types
├── CHANGELOG.md                                [FILE] # Semantic versioned changelog; model checkpoint versions tracked separately from API versions
└── LICENSE                                     [FILE]

---

## Service Dependency Map

                        ┌─────────────────┐
                        │   api_gateway   │  ← External entry point (gRPC + REST + WebSocket)
                        └────────┬────────┘
                                 │ publishes session context
              ┌──────────────────┼──────────────────┐
              │                  │                   │
              ▼                  ▼                   ▼
      ┌──────────────┐  ┌──────────────┐   ┌──────────────────┐
      │  ingestion   │  │  enrollment  │   │  active_liveness  │
      └──────┬───────┘  └──────┬───────┘   └────────┬─────────┘
             │ Kafka            │ Redis               │ Kafka
             │ audio.*          │ embeddings          │ liveness.*
             ▼                  │                     │
      ┌──────────────┐          │                     │
      │   analysis   │◄─────────┘                     │
      └──────┬───────┘   (enrollment embeddings       │
             │ Kafka      for cosine similarity)       │
             │ scores.*                                │
             └───────────────────┐                    │
                                 ▼                    │
                         ┌──────────────┐◄────────────┘
                         │   decision   │
                         └──────┬───────┘
                                │ Redis (AuthDecision)
                                │ Webhook (push)
                                ▼
                         ┌──────────────┐
                         │ api_gateway  │  ← Serves GetDecision RPC / webhook
                         └──────────────┘

      ┌──────────────┐   ┌──────────────┐
      │ threat_intel │   │  forensics   │  ← Standalone; reads from audit store
      └──────────────┘   └──────────────┘
             ▲
             │ fraud feedback (ReportFraud RPC)
      ┌──────────────┐
      │ api_gateway  │
      └──────────────┘

---

## Key Architectural Invariants

| Constraint | Rationale |

| No audio is persisted by default | GDPR/HIPAA compliance; only flagged sessions go to audit bucket, customer-controlled |
| All inter-service communication via Kafka (async) or Redis (sync read) | Decouples services; analysis pod crash does not drop the API response |
| Model weights never loaded from network at inference time | Air-gap safe; weights mounted as volumes at pod startup |
| All model parameters frozen at deployment | No training happens in production; drift is addressed via offline checkpoint rotation only |
| Enrollment anti-spoof guard is non-bypassable | Prevents speaker model poisoning via synthetic enrollment audio |
| Secrets never in git | External Secrets Operator or Sealed Secrets; `.env.example` contains structure only |
| Every service exposes `/health` and `/ready` | Kubernetes liveness and readiness probes required for zero-downtime rollouts |
