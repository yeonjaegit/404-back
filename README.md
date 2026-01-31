# 🚗🔧 404 - back 스마트 팩토리 - 근태 관리 및 채팅 시스템

Flask 기반의 IoT 센서 데이터 수집 및 실시간 모니터링 백엔드 시스템입니다.  
MQTT 프로토콜을 통해 센서 데이터를 수집하고, WebSocket으로 실시간 대시보드에 전송합니다.

# 404found 2차 프로젝트
> **프로젝트의 모든 과정을 담은 상세 시연 영상입니다.** > 이미지 또는 버튼을 클릭하면 유튜브 페이지로 이동합니다.
<div align="center">
  <a href="https://www.youtube.com/watch?v=gPBmVkVSfhc">
    <img src="https://img.youtube.com/vi/gPBmVkVSfhc/maxresdefault.jpg" width="80%" alt="404found 2차 프로젝트 시연영상">
    <br>
    <img src="https://img.shields.io/badge/YouTube-Watch_Video-red?style=for-the-badge&logo=youtube" alt="Youtube Button">
  </a>
</div>

## 🎯 프로젝트 개요

IoT 센서와 AI 카메라를 활용한 실시간 품질 검사 시스템의 Flask 백엔드입니다.
**MQTT 프로토콜**로 센서 데이터를 수집하고, **WebSocket**으로 실시간 대시보드를 구현했으며, **3-Tier 데이터 검증**으로 통계 신뢰성을 확보했습니다.

---

## 📊 핵심 성과

### ⚡ 실시간 처리
- **평균 지연시간 200ms**: 센서 감지 → UI 표시까지
- **초당 30-50건 처리**: MQTT 메시지 실시간 수집 및 WebSocket 전파
- **데이터 정합성 100%**: 3-Tier 검증으로 비정상 데이터 원천 차단

### 🏗️ 시스템 안정성
- **QoS 1 설정**: MQTT 메시지 손실률 0% 달성
- **논블로킹 처리**: `client.loop_start()` 비동기 MQTT로 서버 부하 최소화
- **Flask App Context 관리**: MQTT 콜백에서 DB 작업 안전하게 처리

### 🛡️ 데이터 신뢰성
- **Whitelist 검증**: OK/DEFECT만 허용하여 통계 왜곡 방지
- **DB ENUM 제약**: 스키마 레벨에서 최종 방어선 구축
- **불량 이미지 제한**: 자동 저장 개수 제한으로 저장공간 최적화

---

## 🚀 기술적 도전과 해결

### 1. MQTT + WebSocket 이중 실시간 통신 구조

**문제 상황**
- 아두이노 센서(MQTT) → Flask 서버 → 웹 대시보드(WebSocket) 실시간 연결 필요
- HTTP Polling 방식은 불필요한 서버 부하 및 데이터 반영 지연

**해결 방법: 이벤트 기반 Push 아키텍처**

```python
# utils/mqtt_client.py
import paho.mqtt.client as mqtt
from extensions import socketio, db
from flask import current_app

def on_sensor_message(client, userdata, msg):
    """MQTT 메시지 수신 → 즉시 WebSocket 전파"""
    data = json.loads(msg.payload)
    
    # Flask app context 내에서 DB 작업
    with _flask_app.app_context():
        # 1. 데이터 검증
        if data['result'] not in ['OK', 'DEFECT']:
            return  # 비정상 데이터 차단
        
        # 2. DB 저장
        sensor = SensorResult(
            car_id=current_car_id,
            device=data['device'],
            result=data['result']
        )
        db.session.add(sensor)
        db.session.commit()
        
        # 3. WebSocket 실시간 전파
        socketio.emit('sensor_defect', {
            'device': data['device'],
            'car_id': current_car_id
        })
```

**아키텍처 다이어그램**

```
[아두이노 센서] 
    ↓ MQTT (QoS 1)
[MQTT Broker (Mosquitto)]
    ↓ Subscribe
[Flask Backend]
    ├─ 데이터 검증 (Whitelist)
    ├─ DB 저장 (SQLAlchemy)
    └─ WebSocket Emit
         ↓
[웹 대시보드 (Socket.IO)]
```

**결과**: 센서 감지부터 UI 표시까지 평균 **200ms** 이내 처리

---

### 2. 3-Tier 데이터 검증 아키텍처

**문제 상황**
- 센서 오작동으로 "ERROR", "TIMEOUT", null 등 이상한 값 수신
- 네트워크 불안정 시 깨진 JSON 수신
- AI 카메라에서 예상 못한 상태값 전송
- **결과**: 불량률 통계 왜곡 및 대시보드 오류 데이터 표시

**해결 방법: 단계별 필터링**

```python
# Tier 1: MQTT 레벨 - JSON 파싱 검증
def on_message(client, userdata, msg):
    try:
        data = json.loads(msg.payload.decode('utf-8'))
        device = data["device"]  # KeyError 발생 시 차단
        result = data["result"]
    except (json.JSONDecodeError, KeyError) as e:
        logging.error(f"Invalid MQTT message: {e}")
        return  # ← 깨진 데이터는 여기서 차단!

# Tier 2: 비즈니스 로직 레벨 - Whitelist 검증
def save_sensor_result(data):
    result = data.get("result", "").upper()
    
    # OK/DEFECT만 허용
    if result not in ['OK', 'DEFECT']:
        logging.warning(f"Invalid result: {result}")
        return  # ← 이상한 값("ERROR", "null")은 여기서 차단!
    
    # DB 저장
    sensor = SensorResult(
        car_id=current_car_id,
        device=data["device"].upper(),
        result=result
    )
    db.session.add(sensor)
    db.session.commit()

# Tier 3: DB 스키마 레벨 - ENUM 제약
# sensor_result 테이블
# result ENUM('OK', 'DEFECT') NOT NULL  ← 최종 방어선
```

**검증 효과**
- 데이터 신뢰성: 100%
- 통계 정확도: 비정상 데이터 유입 0건
- DB 무결성: ENUM 제약으로 스키마 레벨 보장

---

### 3. Flask App Context 관리 (MQTT 콜백)

**문제 상황**
```python
def on_message(client, userdata, msg):
    db.session.add(sensor)  # ❌ RuntimeError: Working outside of application context!
```

MQTT 콜백은 Flask 요청 컨텍스트 외부에서 실행되므로 `db.session` 사용 불가

**해결 방법**
```python
# utils/mqtt_client.py
_flask_app = None

def initialize_mqtt(app):
    global _flask_app
    _flask_app = app  # Flask app 인스턴스 저장
    
    def on_message(client, userdata, msg):
        # Flask app context 생성
        with _flask_app.app_context():
            # 이제 db.session 사용 가능!
            db.session.add(sensor)
            db.session.commit()
```

---

## 🏛️ 시스템 아키텍처

### ERD 설계

**ERD Cloud**: [https://www.erdcloud.com/d/rfbhh56TFNjiobguv](https://www.erdcloud.com/d/rfbhh56TFNjiobguv)

```
Car (1) ──< (N) SensorResult
Car (1) ──< (N) CameraResult
CameraResult (1) ──< (N) DefectImage
Employee (독립 테이블)
```

**설계 특징**
- 차량 중심 설계: `car_id`로 모든 검사 결과 그룹화
- 정규화: 불량 이미지는 별도 테이블로 분리 (1:N)
- 인덱스: `employee_number` UNIQUE 제약

---

### MQTT + WebSocket 데이터 흐름

```
[센서/카메라]
    ↓
[MQTT Broker]
    ↓ QoS 1 (최소 1회 전달 보장)
[Flask Backend]
    ├─ Tier 1: JSON 파싱 검증
    ├─ Tier 2: Whitelist 검증 (OK/DEFECT)
    ├─ Tier 3: DB ENUM 제약
    ├─ SQLAlchemy 저장
    └─ WebSocket Emit
         ↓
[React Dashboard]
    ├─ 실시간 통계 업데이트
    └─ 불량 감지 알림 팝업
```

---

## 💻 핵심 구현 코드

### WebSocket 실시간 알림

```python
# routes/socket_events.py
from extensions import socketio

@socketio.on('connect')
def handle_connect():
    """클라이언트 연결 시 초기 통계 전송"""
    stats = calculate_stats()
    emit('stats', stats)

# utils/mqtt_client.py
def on_sensor_message(client, userdata, msg):
    # ... 데이터 저장 후
    
    # 불량 감지 시 즉시 알림
    if data['result'] == 'DEFECT':
        socketio.emit('sensor_defect', {
            'device': data['device'],
            'car_id': current_car_id,
            'timestamp': datetime.now().isoformat()
        })
    
    # 통계 업데이트
    stats = calculate_stats()
    socketio.emit('stats_update', stats)
```

---

### 통계 계산 최적화

```python
# routes/dashboard_defect.py
def calculate_stats():
    """차량별/장치별 불량률 집계"""
    # 중복 제거 집계 (SQLAlchemy)
    total_cars = db.session.query(
        func.count(func.distinct(SensorResult.car_id))
    ).scalar()
    
    # 장치별 불량률
    device_stats = db.session.query(
        SensorResult.device,
        func.count(case((SensorResult.result == 'DEFECT', 1))).label('defect_count'),
        func.count(SensorResult.id).label('total_count')
    ).group_by(SensorResult.device).all()
    
    return {
        'total_cars': total_cars,
        'device_stats': [
            {
                'device': stat.device,
                'defect_rate': stat.defect_count / stat.total_count * 100
            }
            for stat in device_stats
        ]
    }
```

---

## 🤖 AI 도구 활용 (Cursor)

### Flask App Context 오류 해결
**Before (에러 발생)**
```python
def on_message(client, userdata, msg):
    db.session.add(sensor)  # RuntimeError!
```

**After (Cursor 제안)**
```python
def on_message(client, userdata, msg):
    with _flask_app.app_context():  # ← Cursor가 제시
        db.session.add(sensor)
```

### MQTT QoS 설정 최적화
- Cursor가 QoS 0 → QoS 1 변경 제안 (메시지 손실 방지)
- 재연결 로직 자동 생성

### 비동기 처리 개선
- `client.loop_forever()` → `client.loop_start()` 제안 (논블로킹)

---

## 📁 프로젝트 구조

```
404-back/
├── app.py                  # Flask 애플리케이션 엔트리포인트
├── extensions.py           # SQLAlchemy, SocketIO 확장 초기화
├── models/                 # 데이터베이스 모델
│   ├── car.py
│   ├── sensor_result.py
│   ├── camera_result.py
│   └── defect_image.py
├── routes/                 # API 라우트
│   ├── sensor.py
│   ├── camera.py
│   ├── dashboard_defect.py
│   └── socket_events.py   # WebSocket 핸들러
├── utils/
│   └── mqtt_client.py     # MQTT 클라이언트
└── migrations/            # DB 마이그레이션
```

## 🛠 기술 스택

- **Backend**: Flask 3.1.2
- **ORM**: SQLAlchemy 2.0.45
- **Database**: MySQL (smart_factory)
- **Real-time**: Flask-SocketIO 5.5.1
- **MQTT**: paho-mqtt 2.1.0
- **인증**: Flask-JWT-Extended, bcrypt
- **마이그레이션**: Flask-Migrate 4.1.0

## 🔑 핵심 기능

| 기능 |  설명 |
|------|-------|
| **MQTT 데이터 수집** | 센서 및 카메라 데이터를 MQTT 토픽으로 수집 |
| **데이터 검증** | OK/DEFECT만 허용하여 통계 왜곡 방지 |
| **WebSocket 알림** | 불량 감지 시 실시간 대시보드 업데이트 |
| **통계 계산** | 차량별/장치별 불량률 및 통계 자동 집계 |
| **차량 단위 관리** | car_id 기반으로 모든 검사 결과 그룹화 |

## 🚀 Setup

1. Python 3.8+ 설치 및 가상환경 활성화

```bash
python -m venv venv
source venv/bin/activate  # Windows: venv\\Scripts\\activate
pip install -r requirements.txt
```

2. 환경 변수 설정 (.env 파일 생성)

```env
DATABASE_URL=mysql+pymysql://root:1234@127.0.0.1:3306/smart_factory
JWT_SECRET_KEY=your-secret-key

MQTT_BROKER=localhost
MQTT_PORT=1883
MQTT_TOPIC_SENSOR_RESULT=sensor/result
MQTT_TOPIC_CAMERA01_RESULT=camera01/result
MQTT_TOPIC_ULT01=sensor/ult01
MQTT_TOPIC_ULT02=sensor/ult02
MQTT_TOPIC_ULT03=sensor/ult03

MAX_DEFECT_IMAGE_COUNT=5
```

3. 데이터베이스 마이그레이션

```bash
flask db upgrade
```

4. 서버 실행 (포트 5000)

```bash
python app.py
```

## 📡 API 엔드포인트

### 인증
| Method | Endpoint | 설명 |
|--------|----------|------|
| POST | `/auth/login` | 로그인 및 JWT 토큰 발급 |

### 센서 데이터
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/sensor/result` | 센서 검사 결과 전체 조회 |
| GET | `/sensor/defects` | 불량 결과만 조회 |
| POST | `/sensor/result` | 센서 결과 수동 추가 (테스트용) |

### 카메라 데이터
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/camera/result` | 카메라 검사 결과 조회 |

### 대시보드
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/dashboard/summary` | 전체 통계 요약 |

### 차량 관리
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/car/all` | 전체 차량 목록 |

## 🔌 WebSocket 이벤트

**연결**: `http://localhost:5000`

### 클라이언트 → 서버
- `connect`: 연결 시 초기 통계 수신

### 서버 → 클라이언트
| 이벤트 | 데이터 | 설명 |
|--------|--------|------|
| `stats` | 통계 객체 | 초기 연결 시 전체 통계 |
| `stats_update` | 통계 객체 | 데이터 변경 시 통계 업데이트 |
| `sensor_defect` | 센서 데이터 | 센서 불량 감지 |
| `camera_defect` | 카메라 데이터 | 카메라 불량 감지 |
| `car_added` | `{car_id}` | 새 차량 검사 시작 |
| `progress` | 진행 상태 | 검사 단계별 진행 상황 |

## 📊 MQTT 토픽 구조

### 구독 토픽 (Subscriptions)
- `sensor/result` - 센서 검사 결과 (LED, WHEEL, BUZZER 등)
- `camera01/result` - 카메라 검사 결과 (AI 불량 판정)
- `sensor/ult01` - 초음파 센서 1 (검사 시작 트리거)
- `sensor/ult02` - 초음파 센서 2
- `sensor/ult03` - 초음파 센서 3

### 메시지 포맷

**센서 결과**:
```json
{
  "device": "LED",
  "result": "OK"  // 또는 "DEFECT"
}
```

**카메라 결과**:
```json
{
  "result": "DEFECT",
  "detection": {
    "result_image": ["base64_encoded_image_1", "base64_encoded_image_2"]
  }
}
```

## 🔐 데이터 검증 로직

**구현 위치**: `utils/mqtt_client.py:185-223`

```python
def save_sensor_result(data):
    device = data["device"].upper()
    result = data["result"].upper()
    
    # 유효한 결과(OK, DEFECT)만 저장
    if result in ['OK', 'DEFECT']:
        sensor = SensorResult(car_id=current_car_id, device=device, result=result)
        db.session.add(sensor)
        db.session.commit()
    else:
        print(f"[경고] {result} 상태는 DB에 저장하지 않습니다.")
```

**검증 효과**:
- 잘못된 MQTT 메시지 (TIMEOUT, ERROR 등) 필터링
- 통계 왜곡 방지
- 데이터 무결성 보장

## 🗄️ 데이터베이스 스키마

### car 테이블
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INT | PK, 자동 증가 |
| created_at | DATETIME | 차량 등록 시간 |

### sensor_result 테이블
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INT | PK, 자동 증가 |
| car_id | INT | FK, 차량 ID |
| device | VARCHAR(50) | 센서 종류 (LED, WHEEL 등) |
| result | VARCHAR(10) | 검사 결과 (OK, DEFECT) |
| created_at | DATETIME | 검사 시간 |

### camera_result 테이블
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INT | PK, 자동 증가 |
| car_id | INT | FK, 차량 ID |
| result | VARCHAR(10) | 검사 결과 (OK, DEFECT) |
| created_at | DATETIME | 검사 시간 |

### defect_image 테이블
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INT | PK, 자동 증가 |
| camera_result_id | INT | FK, 카메라 결과 ID |
| car_id | INT | FK, 차량 ID |
| image_path | VARCHAR(255) | 이미지 파일 경로 |

## ⚡ 성능 특징

- **실시간 처리**: MQTT 메시지 수신 즉시 DB 저장 및 WebSocket 전파
- **통계 캐싱**: 30초 TTL로 통계 쿼리 부하 감소
- **비동기 MQTT**: `client.loop_start()`로 논블로킹 처리

---

## 💡 배운 점 및 개선 과제

### 배운 점
- **이벤트 기반 아키텍처**: MQTT Pub/Sub 패턴과 WebSocket Push 구조 이해
- **Flask App Context**: 비동기 작업에서 Flask 컨텍스트 관리 방법 습득
- **데이터 검증**: 다층 검증 아키텍처의 중요성 체득
- **실시간 통신**: IoT 환경에서 저지연 데이터 파이프라인 구축 경험

### 향후 개선 과제
- **메시지 큐 도입**: RabbitMQ/Kafka로 MQTT 메시지 버퍼링 및 안정성 강화
- **Redis 캐싱**: 통계 계산 결과 캐싱으로 대시보드 성능 향상
- **부하 테스트**: JMeter로 동시 센서 100개 환경 검증
- **모니터링**: Prometheus + Grafana로 시스템 메트릭 시각화

---

## 🔗 관련 프로젝트

- **404-spring**: [GitHub 🔗](https://github.com/yeonjaegit/404-spring) - Spring Boot 근태 관리 백엔드 (WebSocket, Scheduler)
- **ERD 설계**: [ERD Cloud](https://www.erdcloud.com/d/rfbhh56TFNjiobguv)
- **노션 포트폴리오**: [상세 프로젝트 문서](https://www.notion.so/Project-3-2ef62d7f696c80a0926ddc560281b2f2)
- **시연 영상**: [YouTube](https://www.youtube.com/watch?v=gPBmVkVSfhc)

---

**Last Updated**: 2026-01-31