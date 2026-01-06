# 드럼 믹싱 시뮬레이터 - 확장된 프로젝트 구조 및 라이브러리

사용자 설정 파라미터와 뒤집힘 측정 로직을 포함한 완전한 구조를 제안드립니다.

---

## 1. 확장된 라이브러리 스택

| 라이브러리 | 버전 | 용도 | CDN |
|-----------|------|------|-----|
| **Matter.js** | 0.19.0 | 2D 물리 엔진 | ✅ |
| **Chart.js** | 4.4.1 | 실시간 지표 그래프 | ✅ |
| **Vanilla JS** | - | UI/컨트롤/상태관리 | 내장 |

```html
<!-- CDN Dependencies -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/matter-js/0.19.0/matter.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.1/dist/chart.umd.min.js"></script>
```

---

## 2. 완전한 파라미터 구조

```javascript
const CONFIG = {
    // ═══════════════════════════════════════
    // 1. 드럼 설정 (DRUM)
    // ═══════════════════════════════════════
    drum: {
        radius: 250,           // 드럼 반지름 (px) [100-400]
        rpm: 15,               // 분당 회전수 [1-60]
        wallThickness: 10,     // 벽 두께
        wallSegments: 48,      // 원형 근사 세그먼트 수
    },
    
    // ═══════════════════════════════════════
    // 2. 배플 설정 (BAFFLES)
    // ═══════════════════════════════════════
    baffles: {
        count: 4,              // 배플 개수 [0-12]
        length: 60,            // 배플 길이 (px) [20-120]
        width: 8,              // 배플 두께 (px)
        tiltAngle: 15,         // 기울기 각도 (°) [-45 ~ +45]
        radialOffset: 0,       // 반경 방향 오프셋 (%) [0-50]
                               // 0=벽에 붙음, 50=중간까지
        pattern: 'uniform',    // 배치 패턴: 'uniform', 'alternating', 'clustered'
        customAngles: [],      // 커스텀 배치 시 각도 배열 (°)
    },
    
    // ═══════════════════════════════════════
    // 3. 재료/입자 설정 (PARTICLES)
    // ═══════════════════════════════════════
    particles: {
        count: 150,            // 입자 수 [50-300]
        size: 18,              // 평균 크기 (px) [10-40]
        sizeVariation: 0.2,    // 크기 편차 (0-0.5)
        shape: 'roundedRect',  // 'circle', 'roundedRect', 'polygon'
        aspectRatio: 1.5,      // 가로/세로 비율 (roundedRect용)
        
        // 물성
        density: 0.001,        // 밀도
        friction: 0.3,         // 마찰 계수 [0-1]
        frictionStatic: 0.5,   // 정지 마찰
        restitution: 0.2,      // 반발 계수 [0-1]
        
        // 초기 분포
        initialDistribution: 'half-split', // 'half-split', 'random', 'layered'
    },
    
    // ═══════════════════════════════════════
    // 4. 환경 설정 (ENVIRONMENT)
    // ═══════════════════════════════════════
    environment: {
        gravity: 1.0,          // 중력 스케일 [0.1-3.0]
        timeScale: 1.0,        // 시뮬레이션 속도 [0.1-2.0]
    },
    
    // ═══════════════════════════════════════
    // 5. 측정 설정 (METRICS)
    // ═══════════════════════════════════════
    metrics: {
        updateInterval: 200,   // 지표 업데이트 주기 (ms)
        historyLength: 100,    // 그래프 데이터 포인트 수
        angularBins: 8,        // Mixing Index용 각도 구간 수
        radialBins: 3,         // 반경 방향 구간 수
        flipThreshold: Math.PI, // flip 인정 기준 (rad)
    }
};
```

---

## 3. 뒤집힘 측정 로직 및 스코어 시스템

### 3.1 스코어 체계 개요

```javascript
const SCORING = {
    // ═══════════════════════════════════════
    // 종합 스코어 (0-100)
    // ═══════════════════════════════════════
    overall: 0,
    
    // ═══════════════════════════════════════
    // 개별 지표 (각 0-100)
    // ═══════════════════════════════════════
    metrics: {
        mixingIndex: 0,        // 공간 균일도
        flipScore: 0,          // 뒤집힘 활성도
        coverageScore: 0,      // 영역 커버리지
        motionScore: 0,        // 운동 활성도
    },
    
    // ═══════════════════════════════════════
    // 가중치 (합계 = 1.0)
    // ═══════════════════════════════════════
    weights: {
        mixingIndex: 0.30,
        flipScore: 0.35,       // 뒤집힘이 핵심이므로 가중치 높음
        coverageScore: 0.20,
        motionScore: 0.15,
    }
};
```

### 3.2 핵심 측정 로직

```javascript
/* ═══════════════════════════════════════════════════════════════
   METRIC 1: Mixing Index (공간 균일도) - 0~100
   ═══════════════════════════════════════════════════════════════ */
function calculateMixingIndex() {
    const bins = createAngularBins(CONFIG.metrics.angularBins);
    const targetRatio = 0.5; // A:B = 50:50 목표
    
    // 각 입자를 각도 구간에 배치
    STATE.particles.forEach(particle => {
        const angle = getAngleFromCenter(particle.position);
        const binIdx = Math.floor((angle + Math.PI) / (2 * Math.PI) * bins.length);
        
        if (particle.label === 'A') bins[binIdx].groupA++;
        else bins[binIdx].groupB++;
    });
    
    // 라세리 혼합 지수 (Lacey Mixing Index) 계산
    let sumSquaredDev = 0;
    let validBins = 0;
    
    bins.forEach(bin => {
        const total = bin.groupA + bin.groupB;
        if (total >= 2) {
            const ratio = bin.groupA / total;
            sumSquaredDev += Math.pow(ratio - targetRatio, 2);
            validBins++;
        }
    });
    
    if (validBins === 0) return 0;
    
    const variance = sumSquaredDev / validBins;
    const maxVariance = 0.25; // 완전 분리 시 최대 분산
    const mixingIndex = 1 - Math.sqrt(variance / maxVariance);
    
    return Math.round(Math.max(0, Math.min(100, mixingIndex * 100)));
}

/* ═══════════════════════════════════════════════════════════════
   METRIC 2: Flip Score (뒤집힘 점수) - 0~100
   ═══════════════════════════════════════════════════════════════ */
const FlipTracker = {
    data: new Map(), // particleId -> tracking data
    
    init(particles) {
        this.data.clear();
        particles.forEach(p => {
            this.data.set(p.id, {
                lastAngle: p.angle,
                cumulativeRotation: 0,
                flipCount: 0,
                lastFlipTime: 0,
                flipHistory: [], // 최근 flip 시간 기록
            });
        });
    },
    
    update(particles, currentTime) {
        particles.forEach(p => {
            const tracker = this.data.get(p.id);
            if (!tracker) return;
            
            // 각도 변화 계산 (래핑 처리)
            let deltaAngle = p.angle - tracker.lastAngle;
            while (deltaAngle > Math.PI) deltaAngle -= 2 * Math.PI;
            while (deltaAngle < -Math.PI) deltaAngle += 2 * Math.PI;
            
            tracker.cumulativeRotation += Math.abs(deltaAngle);
            tracker.lastAngle = p.angle;
            
            // flip 판정 (π 라디안 = 180° 회전)
            while (tracker.cumulativeRotation >= CONFIG.metrics.flipThreshold) {
                tracker.flipCount++;
                tracker.cumulativeRotation -= CONFIG.metrics.flipThreshold;
                tracker.lastFlipTime = currentTime;
                tracker.flipHistory.push(currentTime);
                
                // 최근 60초 기록만 유지
                const cutoff = currentTime - 60000;
                tracker.flipHistory = tracker.flipHistory.filter(t => t > cutoff);
            }
        });
    },
    
    getFlipRate() {
        // 분당 평균 뒤집힘 횟수
        let totalRecentFlips = 0;
        this.data.forEach(tracker => {
            totalRecentFlips += tracker.flipHistory.length;
        });
        
        const particleCount = this.data.size || 1;
        return totalRecentFlips / particleCount; // flips/min/particle
    },
    
    getTotalFlips() {
        let total = 0;
        this.data.forEach(tracker => {
            total += tracker.flipCount;
        });
        return total;
    }
};

function calculateFlipScore() {
    const flipRate = FlipTracker.getFlipRate();
    
    // 목표: 분당 10-20회 뒤집힘이 최적
    // 스코어링 곡선 (S-curve)
    const optimalRate = 15;
    const maxRate = 40;
    
    let score;
    if (flipRate <= optimalRate) {
        // 0 ~ optimal: 선형 증가
        score = (flipRate / optimalRate) * 100;
    } else {
        // optimal ~ max: 완만한 감소 (과도한 회전은 약간 감점)
        const excess = (flipRate - optimalRate) / (maxRate - optimalRate);
        score = 100 - (excess * 15); // 최대 15점 감점
    }
    
    return Math.round(Math.max(0, Math.min(100, score)));
}

/* ═══════════════════════════════════════════════════════════════
   METRIC 3: Coverage Score (영역 커버리지) - 0~100
   ═══════════════════════════════════════════════════════════════ */
function calculateCoverageScore() {
    // 드럼을 격자로 나누어 입자가 방문한 영역 추적
    const gridSize = 10;
    const cellSize = (CONFIG.drum.radius * 2) / gridSize;
    const visited = new Set();
    
    STATE.particles.forEach(p => {
        const gridX = Math.floor((p.position.x - STATE.drumCenter.x + CONFIG.drum.radius) / cellSize);
        const gridY = Math.floor((p.position.y - STATE.drumCenter.y + CONFIG.drum.radius) / cellSize);
        
        // 드럼 내부 셀만 카운트
        const distFromCenter = Math.sqrt(
            Math.pow(gridX - gridSize/2, 2) + Math.pow(gridY - gridSize/2, 2)
        );
        if (distFromCenter <= gridSize/2) {
            visited.add(`${gridX},${gridY}`);
        }
    });
    
    // 드럼 내부 총 셀 수 (원형 근사)
    const totalCells = Math.PI * Math.pow(gridSize/2, 2);
    const coverage = visited.size / totalCells;
    
    return Math.round(Math.min(100, coverage * 100));
}

/* ═══════════════════════════════════════════════════════════════
   METRIC 4: Motion Score (운동 활성도) - 0~100
   ═══════════════════════════════════════════════════════════════ */
function calculateMotionScore() {
    let totalSpeed = 0;
    let totalAngularSpeed = 0;
    
    STATE.particles.forEach(p => {
        totalSpeed += Math.sqrt(
            p.velocity.x * p.velocity.x + 
            p.velocity.y * p.velocity.y
        );
        totalAngularSpeed += Math.abs(p.angularVelocity);
    });
    
    const avgSpeed = totalSpeed / STATE.particles.length;
    const avgAngularSpeed = totalAngularSpeed / STATE.particles.length;
    
    // 정규화 (경험적 기준값)
    const speedScore = Math.min(100, (avgSpeed / 5) * 100);
    const angularScore = Math.min(100, (avgAngularSpeed / 0.3) * 100);
    
    return Math.round((speedScore * 0.4 + angularScore * 0.6));
}

/* ═══════════════════════════════════════════════════════════════
   OVERALL SCORE (종합 점수 계산)
   ═══════════════════════════════════════════════════════════════ */
function calculateOverallScore() {
    const metrics = {
        mixingIndex: calculateMixingIndex(),
        flipScore: calculateFlipScore(),
        coverageScore: calculateCoverageScore(),
        motionScore: calculateMotionScore(),
    };
    
    // 가중 평균
    const overall = Math.round(
        metrics.mixingIndex * SCORING.weights.mixingIndex +
        metrics.flipScore * SCORING.weights.flipScore +
        metrics.coverageScore * SCORING.weights.coverageScore +
        metrics.motionScore * SCORING.weights.motionScore
    );
    
    return { overall, ...metrics };
}
```

### 3.3 스코어 등급 시스템

```javascript
const GRADE_SYSTEM = {
    getGrade(score) {
        if (score >= 90) return { grade: 'S', label: '최적', color: '#FFD700' };
        if (score >= 80) return { grade: 'A', label: '우수', color: '#4CAF50' };
        if (score >= 70) return { grade: 'B', label: '양호', color: '#8BC34A' };
        if (score >= 60) return { grade: 'C', label: '보통', color: '#FFC107' };
        if (score >= 50) return { grade: 'D', label: '미흡', color: '#FF9800' };
        return { grade: 'F', label: '부족', color: '#F44336' };
    },
    
    getFlipRateStatus(rate) {
        if (rate < 5) return { status: '정체', suggestion: 'RPM 또는 배플 수 증가 권장' };
        if (rate < 10) return { status: '저조', suggestion: '배플 길이 또는 각도 조정' };
        if (rate <= 20) return { status: '최적', suggestion: '현재 설정 유지' };
        if (rate <= 30) return { status: '활발', suggestion: '양호하나 RPM 감소 검토' };
        return { status: '과다', suggestion: 'RPM 감소 또는 마찰 증가 권장' };
    }
};
```

---

## 4. 완전한 프로젝트 구조

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>🥁 Drum Mixing Simulator</title>
    
    <!-- CDN -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/matter-js/0.19.0/matter.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.1/dist/chart.umd.min.js"></script>
    
    <style>
    /* ═══════════════════════════════════════════════════════════
       SECTION 1: CSS Variables & Reset
       ═══════════════════════════════════════════════════════════ */
    :root {
        --primary: #2196F3;
        --success: #4CAF50;
        --warning: #FFC107;
        --danger: #F44336;
        --dark: #1a1a2e;
        --panel-bg: #16213e;
        --card-bg: #1f2940;
    }
    
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { 
        font-family: 'Segoe UI', system-ui, sans-serif;
        background: var(--dark);
        color: #fff;
        overflow: hidden;
    }

    /* ═══════════════════════════════════════════════════════════
       SECTION 2: Layout (3-Column)
       ═══════════════════════════════════════════════════════════ */
    .app-container {
        display: grid;
        grid-template-columns: 280px 1fr 320px;
        height: 100vh;
        gap: 10px;
        padding: 10px;
    }

    /* ═══════════════════════════════════════════════════════════
       SECTION 3: Control Panel (Left)
       ═══════════════════════════════════════════════════════════ */
    .control-panel {
        background: var(--panel-bg);
        border-radius: 12px;
        padding: 15px;
        overflow-y: auto;
    }
    
    .control-section {
        margin-bottom: 20px;
        padding: 12px;
        background: var(--card-bg);
        border-radius: 8px;
    }
    
    .control-section h3 {
        font-size: 14px;
        color: var(--primary);
        margin-bottom: 12px;
        display: flex;
        align-items: center;
        gap: 8px;
    }
    
    .control-row {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 10px;
    }
    
    .control-row label {
        font-size: 12px;
        color: #aaa;
    }
    
    .control-row input[type="range"] {
        width: 100px;
        accent-color: var(--primary);
    }
    
    .control-row .value {
        font-size: 12px;
        font-weight: bold;
        min-width: 45px;
        text-align: right;
    }

    /* ═══════════════════════════════════════════════════════════
       SECTION 4: Canvas Area (Center)
       ═══════════════════════════════════════════════════════════ */
    .canvas-container {
        background: var(--panel-bg);
        border-radius: 12px;
        display: flex;
        align-items: center;
        justify-content: center;
        position: relative;
    }
    
    #simulation-canvas {
        border-radius: 8px;
    }
    
    .canvas-overlay {
        position: absolute;
        top: 15px;
        left: 15px;
        display: flex;
        gap: 10px;
    }
    
    .overlay-badge {
        background: rgba(0,0,0,0.7);
        padding: 5px 12px;
        border-radius: 20px;
        font-size: 12px;
    }

    /* ═══════════════════════════════════════════════════════════
       SECTION 5: Score Dashboard (Right)
       ═══════════════════════════════════════════════════════════ */
    .score-dashboard {
        background: var(--panel-bg);
        border-radius: 12px;
        padding: 15px;
        display: flex;
        flex-direction: column;
        gap: 15px;
    }
    
    /* Overall Score Card */
    .overall-score-card {
        background: linear-gradient(135deg, #1f2940 0%, #2d3a5a 100%);
        border-radius: 12px;
        padding: 20px;
        text-align: center;
    }
    
    .score-circle {
        width: 120px;
        height: 120px;
        margin: 0 auto 15px;
        position: relative;
    }
    
    .score-value {
        font-size: 42px;
        font-weight: bold;
    }
    
    .score-grade {
        font-size: 18px;
        padding: 4px 16px;
        border-radius: 20px;
        display: inline-block;
    }
    
    /* Individual Metrics */
    .metric-card {
        background: var(--card-bg);
        border-radius: 8px;
        padding: 12px;
    }
    
    .metric-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 8px;
    }
    
    .metric-name {
        font-size: 12px;
        color: #aaa;
    }
    
    .metric-value {
        font-size: 20px;
        font-weight: bold;
    }
    
    .metric-bar {
        height: 6px;
        background: #333;
        border-radius: 3px;
        overflow: hidden;
    }
    
    .metric-bar-fill {
        height: 100%;
        border-radius: 3px;
        transition: width 0.3s ease;
    }
    
    /* Chart Container */
    .chart-container {
        flex: 1;
        min-height: 150px;
    }

    /* ═══════════════════════════════════════════════════════════
       SECTION 6: Buttons
       ═══════════════════════════════════════════════════════════ */
    .btn-group {
        display: flex;
        gap: 8px;
        flex-wrap: wrap;
    }
    
    .btn {
        padding: 8px 16px;
        border: none;
        border-radius: 6px;
        cursor: pointer;
        font-size: 12px;
        font-weight: 600;
        transition: all 0.2s;
    }
    
    .btn-primary { background: var(--primary); color: #fff; }
    .btn-success { background: var(--success); color: #fff; }
    .btn-warning { background: var(--warning); color: #000; }
    .btn-danger { background: var(--danger); color: #fff; }
    
    .btn:hover { transform: translateY(-2px); filter: brightness(1.1); }
    </style>
</head>

<body>
    <div class="app-container">
        <!-- ═══════════════════════════════════════════════════════
             LEFT PANEL: Controls
             ═══════════════════════════════════════════════════════ -->
        <div class="control-panel">
            <h2 style="margin-bottom: 15px;">⚙️ 시뮬레이터 설정</h2>
            
            <!-- 드럼 설정 -->
            <div class="control-section">
                <h3>🥁 드럼 (Drum)</h3>
                <div class="control-row">
                    <label>크기 (반지름)</label>
                    <input type="range" id="drum-radius" min="150" max="350" value="250">
                    <span class="value" id="drum-radius-val">250px</span>
                </div>
                <div class="control-row">
                    <label>회전 속도 (RPM)</label>
                    <input type="range" id="drum-rpm" min="1" max="60" value="15">
                    <span class="value" id="drum-rpm-val">15</span>
                </div>
            </div>
            
            <!-- 배플 설정 -->
            <div class="control-section">
                <h3>📐 배플 (Baffles)</h3>
                <div class="control-row">
                    <label>개수</label>
                    <input type="range" id="baffle-count" min="0" max="12" value="4">
                    <span class="value" id="baffle-count-val">4개</span>
                </div>
                <div class="control-row">
                    <label>길이</label>
                    <input type="range" id="baffle-length" min="20" max="120" value="60">
                    <span class="value" id="baffle-length-val">60px</span>
                </div>
                <div class="control-row">
                    <label>각도 (기울기)</label>
                    <input type="range" id="baffle-tilt" min="-45" max="45" value="15">
                    <span class="value" id="baffle-tilt-val">15°</span>
                </div>
                <div class="control-row">
                    <label>위치 (오프셋)</label>
                    <input type="range" id="baffle-offset" min="0" max="50" value="0">
                    <span class="value" id="baffle-offset-val">0%</span>
                </div>
            </div>
            
            <!-- 재료 설정 -->
            <div class="control-section">
                <h3>🍗 재료 (Particles)</h3>
                <div class="control-row">
                    <label>수량</label>
                    <input type="range" id="particle-count" min="30" max="300" value="150">
                    <span class="value" id="particle-count-val">150개</span>
                </div>
                <div class="control-row">
                    <label>크기</label>
                    <input type="range" id="particle-size" min="10" max="40" value="18">
                    <span class="value" id="particle-size-val">18px</span>
                </div>
                <div class="control-row">
                    <label>마찰</label>
                    <input type="range" id="particle-friction" min="0" max="100" value="30">
                    <span class="value" id="particle-friction-val">0.3</span>
                </div>
                <div class="control-row">
                    <label>탄성</label>
                    <input type="range" id="particle-restitution" min="0" max="100" value="20">
                    <span class="value" id="particle-restitution-val">0.2</span>
                </div>
            </div>
            
            <!-- 환경 설정 -->
            <div class="control-section">
                <h3>🌍 환경</h3>
                <div class="control-row">
                    <label>중력</label>
                    <input type="range" id="gravity" min="10" max="300" value="100">
                    <span class="value" id="gravity-val">1.0g</span>
                </div>
                <div class="control-row">
                    <label>속도</label>
                    <input type="range" id="time-scale" min="10" max="200" value="100">
                    <span class="value" id="time-scale-val">1.0x</span>
                </div>
            </div>
            
            <!-- 버튼 그룹 -->
            <div class="btn-group">
                <button class="btn btn-primary" id="btn-reset">🔄 리셋</button>
                <button class="btn btn-success" id="btn-rebuild">🔨 재구성</button>
                <button class="btn btn-warning" id="btn-randomize">🎲 랜덤</button>
                <button class="btn btn-danger" id="btn-pause">⏸️ 일시정지</button>
            </div>
        </div>
        
        <!-- ═══════════════════════════════════════════════════════
             CENTER: Canvas
             ═══════════════════════════════════════════════════════ -->
        <div class="canvas-container">
            <div class="canvas-overlay">
                <span class="overlay-badge" id="fps-badge">FPS: 60</span>
                <span class="overlay-badge" id="time-badge">⏱️ 0:00</span>
            </div>
            <canvas id="simulation-canvas"></canvas>
        </div>
        
        <!-- ═══════════════════════════════════════════════════════
             RIGHT PANEL: Scores & Metrics
             ═══════════════════════════════════════════════════════ -->
        <div class="score-dashboard">
            <h2>📊 성능 분석</h2>
            
            <!-- Overall Score -->
            <div class="overall-score-card">
                <div class="score-circle">
                    <canvas id="score-gauge"></canvas>
                </div>
                <div class="score-value" id="overall-score">0</div>
                <div class="score-grade" id="score-grade" style="background: #333;">측정중</div>
                <div style="margin-top: 10px; font-size: 12px; color: #aaa;" id="score-suggestion">
                    시뮬레이션을 시작하세요
                </div>
            </div>
            
            <!-- Flip Rate (핵심 지표) -->
            <div class="metric-card" style="border-left: 3px solid var(--primary);">
                <div class="metric-header">
                    <span class="metric-name">🔄 뒤집힘 (Flip Rate)</span>
                    <span class="metric-value" id="flip-rate">0</span>
                </div>
                <div style="font-size: 11px; color: #888;">회/분/개당</div>
                <div class="metric-bar">
                    <div class="metric-bar-fill" id="flip-bar" 
                         style="width: 0%; background: var(--primary);"></div>
                </div>
                <div style="margin-top: 8px; font-size: 11px;" id="flip-status">
                    상태: 대기중
                </div>
            </div>
            
            <!-- Mixing Index -->
            <div class="metric-card">
                <div class="metric-header">
                    <span class="metric-name">🎯 혼합도 (Mixing Index)</span>
                    <span class="metric-value" id="mixing-index">0</span>
                </div>
                <div class="metric-bar">
                    <div class="metric-bar-fill" id="mixing-bar" 
                         style="width: 0%; background: var(--success);"></div>
                </div>
            </div>
            
            <!-- Coverage -->
            <div class="metric-card">
                <div class="metric-header">
                    <span class="metric-name">📍 커버리지</span>
                    <span class="metric-value" id="coverage">0</span>
                </div>
                <div class="metric-bar">
                    <div class="metric-bar-fill" id="coverage-bar" 
                         style="width: 0%; background: var(--warning);"></div>
                </div>
            </div>
            
            <!-- Motion -->
            <div class="metric-card">
                <div class="metric-header">
                    <span class="metric-name">⚡ 운동량</span>
                    <span class="metric-value" id="motion">0</span>
                </div>
                <div class="metric-bar">
                    <div class="metric-bar-fill" id="motion-bar" 
                         style="width: 0%; background: #9C27B0;"></div>
                </div>
            </div>
            
            <!-- Time Series Chart -->
            <div class="chart-container">
                <canvas id="metrics-chart"></canvas>
            </div>
            
            <!-- Export Buttons -->
            <div class="btn-group">
                <button class="btn btn-primary" id="btn-export-csv">📥 CSV 내보내기</button>
                <button class="btn btn-success" id="btn-save-preset">💾 설정 저장</button>
            </div>
        </div>
    </div>

    <script>
    /* ═══════════════════════════════════════════════════════════════════════
       ███████╗███████╗ ██████╗████████╗██╗ ██████╗ ███╗   ██╗     ██╗
       ██╔════╝██╔════╝██╔════╝╚══██╔══╝██║██╔═══██╗████╗  ██║    ███║
       ███████╗█████╗  ██║        ██║   ██║██║   ██║██╔██╗ ██║    ╚██║
       ╚════██║██╔══╝  ██║        ██║   ██║██║   ██║██║╚██╗██║     ██║
       ███████║███████╗╚██████╗   ██║   ██║╚██████╔╝██║ ╚████║     ██║
       ╚══════╝╚══════╝ ╚═════╝   ╚═╝   ╚═╝ ╚═════╝ ╚═╝  ╚═══╝     ╚═╝
       CONFIGURATION & STATE
       ═══════════════════════════════════════════════════════════════════════ */
    
    const { Engine, Render, Runner, Bodies, Body, Composite, Events, Vector } = Matter;
    
    // Global Configuration
    const CONFIG = {
        drum: { radius: 250, rpm: 15, wallSegments: 48 },
        baffles: { count: 4, length: 60, width: 8, tilt: 15, offset: 0 },
        particles: { count: 150, size: 18, friction: 0.3, restitution: 0.2 },
        environment: { gravity: 1.0, timeScale: 1.0 },
        metrics: { updateInterval: 200, angularBins: 8 }
    };
    
    // Global State
    const STATE = {
        engine: null,
        render: null,
        runner: null,
        drumComposite: null,
        particles: [],
        drumCenter: { x: 0, y: 0 },
        drumAngle: 0,
        isPaused: false,
        startTime: Date.now(),
        metrics: { overall: 0, flipScore: 0, mixingIndex: 0, coverage: 0, motion: 0 },
        metricsHistory: []
    };

    /* ═══════════════════════════════════════════════════════════════════════
       ███████╗███████╗ ██████╗████████╗██╗ ██████╗ ███╗   ██╗    ██████╗ 
       ██╔════╝██╔════╝██╔════╝╚══██╔══╝██║██╔═══██╗████╗  ██║    ╚════██╗
       ███████╗█████╗  ██║        ██║   ██║██║   ██║██╔██╗ ██║     █████╔╝
       ╚════██║██╔══╝  ██║        ██║   ██║██║   ██║██║╚██╗██║    ██╔═══╝ 
       ███████║███████╗╚██████╗   ██║   ██║╚██████╔╝██║ ╚████║    ███████╗
       ╚══════╝╚══════╝ ╚═════╝   ╚═╝   ╚═╝ ╚═════╝ ╚═╝  ╚═══╝    ╚══════╝
       FLIP TRACKING SYSTEM
       ═══════════════════════════════════════════════════════════════════════ */
    
    const FlipTracker = {
        data: new Map(),
        
        init(particles) {
            this.data.clear();
            particles.forEach(p => {
                this.data.set(p.id, {
                    lastAngle: p.angle,
                    cumRotation: 0,
                    flipCount: 0,
                    flipHistory: []
                });
            });
        },
        
        update(particles, currentTime) {
            particles.forEach(p => {
                const t = this.data.get(p.id);
                if (!t) return;
                
                let delta = p.angle - t.lastAngle;
                while (delta > Math.PI) delta -= 2 * Math.PI;
                while (delta < -Math.PI) delta += 2 * Math.PI;
                
                t.cumRotation += Math.abs(delta);
                t.lastAngle = p.angle;
                
                while (t.cumRotation >= Math.PI) {
                    t.flipCount++;
                    t.cumRotation -= Math.PI;
                    t.flipHistory.push(currentTime);
                    t.flipHistory = t.flipHistory.filter(time => time > currentTime - 60000);
                }
            });
        },
        
        getFlipRate() {
            let total = 0;
            this.data.forEach(t => { total += t.flipHistory.length; });
            return this.data.size > 0 ? (total / this.data.size).toFixed(1) : 0;
        },
        
        getTotalFlips() {
            let total = 0;
            this.data.forEach(t => { total += t.flipCount; });
            return total;
        }
    };

    /* ═══════════════════════════════════════════════════════════════════════
       ███████╗███████╗ ██████╗████████╗██╗ ██████╗ ███╗   ██╗    ██████╗ 
       ██╔════╝██╔════╝██╔════╝╚══██╔══╝██║██╔═══██╗████╗  ██║    ╚════██╗
       ███████╗█████╗  ██║        ██║   ██║██║   ██║██╔██╗ ██║     █████╔╝
       ╚════██║██╔══╝  ██║        ██║   ██║██║   ██║██║╚██╗██║     ╚═══██╗
       ███████║███████╗╚██████╗   ██║   ██║╚██████╔╝██║ ╚████║    ██████╔╝
       ╚══════╝╚══════╝ ╚═════╝   ╚═╝   ╚═╝ ╚═════╝ ╚═╝  ╚═══╝    ╚═════╝ 
       METRICS CALCULATION
       ═══════════════════════════════════════════════════════════════════════ */
    
    function calcMixingIndex() {
        const bins = Array(CONFIG.metrics.angularBins).fill(null).map(() => ({ a: 0, b: 0 }));
        
        STATE.particles.forEach(p => {
            const dx = p.position.x - STATE.drumCenter.x;
            const dy = p.position.y - STATE.drumCenter.y;
            const angle = Math.atan2(dy, dx);
            const idx = Math.floor((angle + Math.PI) / (2 * Math.PI) * bins.length) % bins.length;
            
            if (p.label === 'A') bins[idx].a++;
            else bins[idx].b++;
        });
        
        let variance = 0, validBins = 0;
        bins.forEach(bin => {
            const total = bin.a + bin.b;
            if (total >= 2) {
                variance += Math.pow(bin.a / total - 0.5, 2);
                validBins++;
            }
        });
        
        if (validBins === 0) return 0;
        return Math.round(Math.max(0, (1 - Math.sqrt(variance / validBins) * 2)) * 100);
    }
    
    function calcFlipScore() {
        const rate = parseFloat(FlipTracker.getFlipRate());
        const optimal = 15, max = 40;
        
        let score;
        if (rate <= optimal) score = (rate / optimal) * 100;
        else score = 100 - ((rate - optimal) / (max - optimal)) * 15;
        
        return Math.round(Math.max(0, Math.min(100, score)));
    }
    
    function calcCoverage() {
        const gridSize = 10;
        const visited = new Set();
        
        STATE.particles.forEach(p => {
            const gx = Math.floor((p.position.x - STATE.drumCenter.x + CONFIG.drum.radius) / (CONFIG.drum.radius * 2 / gridSize));
            const gy = Math.floor((p.position.y - STATE.drumCenter.y + CONFIG.drum.radius) / (CONFIG.drum.radius * 2 / gridSize));
            visited.add(`${gx},${gy}`);
        });
        
        return Math.round(Math.min(100, visited.size / (Math.PI * 25) * 100));
    }
    
    function calcMotion() {
        let totalSpeed = 0, totalAngular = 0;
        
        STATE.particles.forEach(p => {
            totalSpeed += Math.sqrt(p.velocity.x ** 2 + p.velocity.y ** 2);
            totalAngular += Math.abs(p.angularVelocity);
        });
        
        const avgSpeed = totalSpeed / STATE.particles.length;
        const avgAngular = totalAngular / STATE.particles.length;
        
        return Math.round((Math.min(100, avgSpeed / 5 * 100) * 0.4 + Math.min(100, avgAngular / 0.3 * 100) * 0.6));
    }
    
    function calcOverall() {
        return Math.round(
            STATE.metrics.mixingIndex * 0.30 +
            STATE.metrics.flipScore * 0.35 +
            STATE.metrics.coverage * 0.20 +
            STATE.metrics.motion * 0.15
        );
    }

    /* ═══════════════════════════════════════════════════════════════════════
       ███████╗███████╗ ██████╗████████╗██╗ ██████╗ ███╗   ██╗    ██╗  ██╗
       ██╔════╝██╔════╝██╔════╝╚══██╔══╝██║██╔═══██╗████╗  ██║    ██║  ██║
       ███████╗█████╗  ██║        ██║   ██║██║   ██║██╔██╗ ██║    ███████║
       ╚════██║██╔══╝  ██║        ██║   ██║██║   ██║██║╚██╗██║    ╚════██║
       ███████║███████╗╚██████╗   ██║   ██║╚██████╔╝██║ ╚████║         ██║
       ╚══════╝╚══════╝ ╚═════╝   ╚═╝   ╚═╝ ╚═════╝ ╚═╝  ╚═══╝         ╚═╝
       PHYSICS WORLD SETUP
       ═══════════════════════════════════════════════════════════════════════ */
    
    function createDrumWalls() {
        const walls = [];
        const segments = CONFIG.drum.wallSegments;
        const r = CONFIG.drum.radius;
        
        for (let i = 0; i < segments; i++) {
            const angle = (2 * Math.PI * i) / segments;
            const nextAngle = (2 * Math.PI * (i + 1)) / segments;
            const midAngle = (angle + nextAngle) / 2;
            
            const x = STATE.drumCenter.x + Math.cos(midAngle) * r;
            const y = STATE.drumCenter.y + Math.sin(midAngle) * r;
            const length = 2 * r * Math.sin(Math.PI / segments) + 5;
            
            const wall = Bodies.rectangle(x, y, length, 12, {
                isStatic: true,
                angle: midAngle + Math.PI / 2,
                render: { fillStyle: '#445' },
                friction: 0.8
            });
            walls.push(wall);
        }
        return walls;
    }
    
    function createBaffles() {
        const baffles = [];
        const count = CONFIG.baffles.count;
        if (count === 0) return baffles;
        
        const r = CONFIG.drum.radius;
        const len = CONFIG.baffles.length;
        const tilt = CONFIG.baffles.tilt * Math.PI / 180;
        const offset = CONFIG.baffles.offset / 100;
        
        for (let i = 0; i < count; i++) {
            const angle = (2 * Math.PI * i) / count;
            const baffleR = r - len / 2 - (r * offset * 0.5);
            
            const x = STATE.drumCenter.x + Math.cos(angle) * baffleR;
            const y = STATE.drumCenter.y + Math.sin(angle) * baffleR;
            
            const baffle = Bodies.rectangle(x, y, CONFIG.baffles.width, len, {
                isStatic: true,
                angle: angle + Math.PI / 2 + tilt,
                render: { fillStyle: '#E91E63' },
                friction: 0.6
            });
            baffles.push(baffle);
        }
        return baffles;
    }
    
    function createParticles() {
        const particles = [];
        const count = CONFIG.particles.count;
        const size = CONFIG.particles.size;
        const r = CONFIG.drum.radius - CONFIG.baffles.length - 30;
        
        for (let i = 0; i < count; i++) {
            const angle = Math.random() * 2 * Math.PI;
            const dist = Math.random() * r * 0.8;
            const x = STATE.drumCenter.x + Math.cos(angle) * dist;
            const y = STATE.drumCenter.y + Math.sin(angle) * dist;
            
            const isGroupA = i < count / 2;
            const variation = 1 + (Math.random() - 0.5) * 0.3;
            
            const particle = Bodies.rectangle(x, y, size * 1.4 * variation, size * variation, {
                chamfer: { radius: 4 },
                friction: CONFIG.particles.friction,
                frictionStatic: CONFIG.particles.friction * 1.5,
                restitution: CONFIG.particles.restitution,
                density: 0.001,
                label: isGroupA ? 'A' : 'B',
                render: { fillStyle: isGroupA ? '#FF6B6B' : '#4ECDC4' }
            });
            particles.push(particle);
        }
        return particles;
    }
    
    function buildWorld() {
        // Clear existing
        if (STATE.engine) {
            Composite.clear(STATE.engine.world, false);
        }
        
        // Create drum assembly
        const walls = createDrumWalls();
        const baffles = createBaffles();
        STATE.drumComposite = Composite.create({ bodies: [...walls, ...baffles] });
        
        // Create particles
        STATE.particles = createParticles();
        
        // Add to world
        Composite.add(STATE.engine.world, STATE.drumComposite);
        Composite.add(STATE.engine.world, STATE.particles);
        
        // Init flip tracker
        FlipTracker.init(STATE.particles);
        STATE.drumAngle = 0;
        STATE.startTime = Date.now();
    }

    /* ═══════════════════════════════════════════════════════════════════════
       ███████╗███████╗ ██████╗████████╗██╗ ██████╗ ███╗   ██╗    ███████╗
       ██╔════╝██╔════╝██╔════╝╚══██╔══╝██║██╔═══██╗████╗  ██║    ██╔════╝
       ███████╗█████╗  ██║        ██║   ██║██║   ██║██╔██╗ ██║    ███████╗
       ╚════██║██╔══╝  ██║        ██║   ██║██║   ██║██║╚██╗██║    ╚════██║
       ███████║███████╗╚██████╗   ██║   ██║╚██████╔╝██║ ╚████║    ███████║
       ╚══════╝╚══════╝ ╚═════╝   ╚═╝   ╚═╝ ╚═════╝ ╚═╝  ╚═══╝    ╚══════╝
       UI & DISPLAY UPDATE
       ═══════════════════════════════════════════════════════════════════════ */
    
    let metricsChart = null;
    
    function initChart() {
        const ctx = document.getElementById('metrics-chart').getContext('2d');
        metricsChart = new Chart(ctx, {
            type: 'line',
            data: {
                labels: [],
                datasets: [
                    { label: '종합', data: [], borderColor: '#FFD700', tension: 0.3, pointRadius: 0 },
                    { label: '뒤집힘', data: [], borderColor: '#2196F3', tension: 0.3, pointRadius: 0 },
                    { label: '혼합도', data: [], borderColor: '#4CAF50', tension: 0.3, pointRadius: 0 }
                ]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                scales: {
                    y: { min: 0, max: 100, grid: { color: '#333' } },
                    x: { display: false }
                },
                plugins: { legend: { labels: { color: '#aaa', font: { size: 10 } } } }
            }
        });
    }
    
    function updateDisplay() {
        // Update metrics
        STATE.metrics.mixingIndex = calcMixingIndex();
        STATE.metrics.flipScore = calcFlipScore();
        STATE.metrics.coverage = calcCoverage();
        STATE.metrics.motion = calcMotion();
        STATE.metrics.overall = calcOverall();
        
        const flipRate = FlipTracker.getFlipRate();
        
        // Update DOM
        document.getElementById('overall-score').textContent = STATE.metrics.overall;
        document.getElementById('flip-rate').textContent = flipRate;
        document.getElementById('mixing-index').textContent = STATE.metrics.mixingIndex;
        document.getElementById('coverage').textContent = STATE.metrics.coverage;
        document.getElementById('motion').textContent = STATE.metrics.motion;
        
        // Update bars
        document.getElementById('flip-bar').style.width = `${Math.min(100, flipRate * 5)}%`;
        document.getElementById('mixing-bar').style.width = `${STATE.metrics.mixingIndex}%`;
        document.getElementById('coverage-bar').style.width = `${STATE.metrics.coverage}%`;
        document.getElementById('motion-bar').style.width = `${STATE.metrics.motion}%`;
        
        // Grade
        const grade = getGrade(STATE.metrics.overall);
        const gradeEl = document.getElementById('score-grade');
        gradeEl.textContent = `${grade.grade} - ${grade.label}`;
        gradeEl.style.background = grade.color;
        
        // Flip status
        const flipStatus = getFlipStatus(parseFloat(flipRate));
        document.getElementById('flip-status').innerHTML = `상태: <b>${flipStatus.status}</b> | ${flipStatus.suggestion}`;
        document.getElementById('score-suggestion').textContent = flipStatus.suggestion;
        
        // Update chart
        if (metricsChart) {
            const now = Math.floor((Date.now() - STATE.startTime) / 1000);
            metricsChart.data.labels.push(now + 's');
            metricsChart.data.datasets[0].data.push(STATE.metrics.overall);
            metricsChart.data.datasets[1].data.push(STATE.metrics.flipScore);
            metricsChart.data.datasets[2].data.push(STATE.metrics.mixingIndex);
            
            if (metricsChart.data.labels.length > 60) {
                metricsChart.data.labels.shift();
                metricsChart.data.datasets.forEach(ds => ds.data.shift());
            }
            metricsChart.update('none');
        }
        
        // Time badge
        const elapsed = Math.floor((Date.now() - STATE.startTime) / 1000);
        document.getElementById('time-badge').textContent = `⏱️ ${Math.floor(elapsed/60)}:${String(elapsed%60).padStart(2,'0')}`;
    }
    
    function getGrade(score) {
        if (score >= 90) return { grade: 'S', label: '최적', color: '#FFD700' };
        if (score >= 80) return { grade: 'A', label: '우수', color: '#4CAF50' };
        if (score >= 70) return { grade: 'B', label: '양호', color: '#8BC34A' };
        if (score >= 60) return { grade: 'C', label: '보통', color: '#FFC107' };
        if (score >= 50) return { grade: 'D', label: '미흡', color: '#FF9800' };
        return { grade: 'F', label: '부족', color: '#F44336' };
    }
    
    function getFlipStatus(rate) {
        if (rate < 3) return { status: '⚪ 정체', suggestion: 'RPM 또는 배플 수 증가 권장' };
        if (rate < 8) return { status: '🟡 저조', suggestion: '배플 길이/각도 조정 필요' };
        if (rate <= 18) return { status: '🟢 최적', suggestion: '현재 설정 유지 권장' };
        if (rate <= 28) return { status: '🔵 활발', suggestion: 'RPM 감소 검토' };
        return { status: '🔴 과다', suggestion: 'RPM 감소 또는 마찰 증가' };
    }

    /* ═══════════════════════════════════════════════════════════════════════
       ███████╗███████╗ ██████╗████████╗██╗ ██████╗ ███╗   ██╗     ██████╗ 
       ██╔════╝██╔════╝██╔════╝╚══██╔══╝██║██╔═══██╗████╗  ██║    ██╔════╝ 
       ███████╗█████╗  ██║        ██║   ██║██║   ██║██╔██╗ ██║    ███████╗ 
       ╚════██║██╔══╝  ██║        ██║   ██║██║   ██║██║╚██╗██║    ██╔═══██╗
       ███████║███████╗╚██████╗   ██║   ██║╚██████╔╝██║ ╚████║    ╚██████╔╝
       ╚══════╝╚══════╝ ╚═════╝   ╚═╝   ╚═╝ ╚═════╝ ╚═╝  ╚═══╝     ╚═════╝ 
       SIMULATION LOOP & CONTROLS
       ═══════════════════════════════════════════════════════════════════════ */
    
    let lastFpsTime = 0, frameCount = 0;
    
    function gameLoop(timestamp) {
        if (!STATE.isPaused) {
            // Rotate drum
            const angularVel = (CONFIG.drum.rpm * 2 * Math.PI) / 60 / 60;
            Composite.rotate(STATE.drumComposite, angularVel, STATE.drumCenter);
            STATE.drumAngle += angularVel;
            
            // Update flip tracker
            FlipTracker.update(STATE.particles, Date.now());
        }
        
        // FPS counter
        frameCount++;
        if (timestamp - lastFpsTime >= 1000) {
            document.getElementById('fps-badge').textContent = `FPS: ${frameCount}`;
            frameCount = 0;
            lastFpsTime = timestamp;
        }
        
        requestAnimationFrame(gameLoop);
    }
    
    function bindControls() {
        // Drum controls
        bindSlider('drum-radius', 'drum-radius-val', v => `${v}px`, v => { CONFIG.drum.radius = parseInt(v); });
        bindSlider('drum-rpm', 'drum-rpm-val', v => v, v => { CONFIG.drum.rpm = parseInt(v); });
        
        // Baffle controls
        bindSlider('baffle-count', 'baffle-count-val', v => `${v}개`, v => { CONFIG.baffles.count = parseInt(v); });
        bindSlider('baffle-length', 'baffle-length-val', v => `${v}px`, v => { CONFIG.baffles.length = parseInt(v); });
        bindSlider('baffle-tilt', 'baffle-tilt-val', v => `${v}°`, v => { CONFIG.baffles.tilt = parseInt(v); });
        bindSlider('baffle-offset', 'baffle-offset-val', v => `${v}%`, v => { CONFIG.baffles.offset = parseInt(v); });
        
        // Particle controls
        bindSlider('particle-count', 'particle-count-val', v => `${v}개`, v => { CONFIG.particles.count = parseInt(v); });
        bindSlider('particle-size', 'particle-size-val', v => `${v}px`, v => { CONFIG.particles.size = parseInt(v); });
        bindSlider('particle-friction', 'particle-friction-val', v => (v/100).toFixed(1), v => { CONFIG.particles.friction = v/100; });
        bindSlider('particle-restitution', 'particle-restitution-val', v => (v/100).toFixed(1), v => { CONFIG.particles.restitution = v/100; });
        
        // Environment controls
        bindSlider('gravity', 'gravity-val', v => `${(v/100).toFixed(1)}g`, v => {
            CONFIG.environment.gravity = v / 100;
            STATE.engine.gravity.y = CONFIG.environment.gravity;
        });
        bindSlider('time-scale', 'time-scale-val', v => `${(v/100).toFixed(1)}x`, v => {
            CONFIG.environment.timeScale = v / 100;
            STATE.engine.timing.timeScale = CONFIG.environment.timeScale;
        });
        
        // Buttons
        document.getElementById('btn-reset').onclick = () => { buildWorld(); };
        document.getElementById('btn-rebuild').onclick = () => { buildWorld(); };
        document.getElementById('btn-randomize').onclick = () => {
            document.getElementById('baffle-count').value = Math.floor(Math.random() * 10) + 2;
            document.getElementById('baffle-tilt').value = Math.floor(Math.random() * 90) - 45;
            document.getElementById('drum-rpm').value = Math.floor(Math.random() * 40) + 5;
            // Trigger change events
            ['baffle-count', 'baffle-tilt', 'drum-rpm'].forEach(id => {
                document.getElementById(id).dispatchEvent(new Event('input'));
            });
            buildWorld();
        };
        document.getElementById('btn-pause').onclick = function() {
            STATE.isPaused = !STATE.isPaused;
            this.textContent = STATE.isPaused ? '▶️ 재생' : '⏸️ 일시정지';
        };
        
        // Export
        document.getElementById('btn-export-csv').onclick = exportCSV;
        document.getElementById('btn-save-preset').onclick = savePreset;
    }
    
    function bindSlider(sliderId, valueId, formatter, setter) {
        const slider = document.getElementById(sliderId);
        const valueEl = document.getElementById(valueId);
        
        slider.addEventListener('input', () => {
            valueEl.textContent = formatter(slider.value);
            setter(slider.value);
        });
    }
    
    function exportCSV() {
        let csv = 'Time(s),Overall,FlipScore,MixingIndex,Coverage,Motion,FlipRate\n';
        const elapsed = Math.floor((Date.now() - STATE.startTime) / 1000);
        csv += `${elapsed},${STATE.metrics.overall},${STATE.metrics.flipScore},${STATE.metrics.mixingIndex},${STATE.metrics.coverage},${STATE.metrics.motion},${FlipTracker.getFlipRate()}\n`;
        
        const blob = new Blob([csv], { type: 'text/csv' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `drum_sim_${Date.now()}.csv`;
        a.click();
    }
    
    function savePreset() {
        const preset = JSON.stringify(CONFIG, null, 2);
        const blob = new Blob([preset], { type: 'application/json' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `preset_${Date.now()}.json`;
        a.click();
    }

    /* ═══════════════════════════════════════════════════════════════════════
       ██╗███╗   ██╗██╗████████╗
       ██║████╗  ██║██║╚══██╔══╝
       ██║██╔██╗ ██║██║   ██║   
       ██║██║╚██╗██║██║   ██║   
       ██║██║ ╚████║██║   ██║   
       ╚═╝╚═╝  ╚═══╝╚═╝   ╚═╝   
       INITIALIZATION
       ═══════════════════════════════════════════════════════════════════════ */
    
    function init() {
        const canvas = document.getElementById('simulation-canvas');
        const container = document.querySelector('.canvas-container');
        
        const width = container.clientWidth - 30;
        const height = container.clientHeight - 30;
        canvas.width = width;
        canvas.height = height;
        
        STATE.drumCenter = { x: width / 2, y: height / 2 };
        
        // Create engine
        STATE.engine = Engine.create({
            gravity: { x: 0, y: CONFIG.environment.gravity }
        });
        
        // Create renderer
        STATE.render = Render.create({
            canvas: canvas,
            engine: STATE.engine,
            options: {
                width: width,
                height: height,
                wireframes: false,
                background: '#0f0f23'
            }
        });
        
        // Create runner
        STATE.runner = Runner.create();
        Runner.run(STATE.runner, STATE.engine);
        Render.run(STATE.render);
        
        // Build world
        buildWorld();
        
        // Init chart
        initChart();
        
        // Bind controls
        bindControls();
        
        // Metrics update interval
        setInterval(updateDisplay, CONFIG.metrics.updateInterval);
        
        // Start game loop
        requestAnimationFrame(gameLoop);
        
        console.log('🥁 Drum Mixing Simulator initialized!');
    }
    
    // Start when DOM ready
    window.addEventListener('DOMContentLoaded', init);
    </script>
</body>
</html>
```

---

## 5. 스코어 시스템 요약

### 5.1 4가지 핵심 지표

| 지표 | 가중치 | 측정 방법 | 최적 범위 |
|------|--------|----------|----------|
| **Flip Score** | 35% | 입자별 π rad 회전 누적 카운트 | 10-20 회/분/개 |
| **Mixing Index** | 30% | θ-bin 기반 A/B 분포 균일도 | 80% 이상 |
| **Coverage** | 20% | 10x10 격자 방문 영역 비율 | 70% 이상 |
| **Motion Score** | 15% | 평균 속도 + 각속도 | 적정 활성 |

### 5.2 등급 체계

| 점수 | 등급 | 판정 |
|------|------|------|
| 90-100 | S | 최적 - 설계 유지 |
| 80-89 | A | 우수 |
| 70-79 | B | 양호 |
| 60-69 | C | 보통 - 조정 권장 |
| 50-59 | D | 미흡 |
| 0-49 | F | 부족 - 재설계 필요 |

---

## 6. 사용 가이드

```
1. 파일을 drum-simulator.html로 저장
2. 브라우저에서 열기 (Chrome/Edge 권장)
3. 좌측 패널에서 파라미터 조정:
   - 드럼: 크기, RPM
   - 배플: 개수, 길이, 각도, 위치
   - 재료: 수량, 크기, 마찰, 탄성
4. [재구성] 버튼으로 변경사항 적용
5. 우측 패널에서 실시간 스코어 확인
6. [CSV 내보내기]로 결과 저장
```

이 구조로 PRD의 모든 요구사항과 확장된 스코어링 시스템을 단일 HTML 파일로 구현할 수 있습니다. 바로 실행해보시겠습니까?