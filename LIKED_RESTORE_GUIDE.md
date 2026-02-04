# 복원된 Liked 리스트 유지 가이드 (코드 수정 없음)

## 📌 문제 분석

복원 후 **YouTube와의 자동 동기화(YtmSync)** 기능이 실행되면서 liked 리스트가 삭제됩니다.

**발생 메커니즘:**
```
복원 완료
  ↓
앱 시작 → MainActivity → AutoPlaylistScreen 렌더링
  ↓
LaunchedEffect(Unit) 실행 (SyncUtils.kt#L217)
  ↓
ytmSync == true → syncLikedSongs() 호출
  ↓
YouTube 계정의 liked 플레이리스트 조회 (API)
  ↓
로컬 liked 곡 vs 원격 liked 곡 비교
  ↓
로컬에만 있는 곡 → liked = false로 변경 (삭제됨)
```

---

## ✅ 해결 방법 (코드 수정 없음)

### **방법 1: 복원 전에 동기화 비활성화** ⭐⭐⭐ (가장 권장)

**단계:**
1. 앱 설정 → 계정 설정 → "YouTube와 동기화" 토글 **OFF**
2. 백업 파일 생성
3. 기존 데이터 삭제
4. 백업 파일 복원
5. 앱이 자동 동기화를 시도하지 않음 (YtmSyncKey = false)
6. 복원 완료 후 필요시 다시 **ON**

**장점:**
- ✅ 가장 안전함
- ✅ liked 리스트 100% 보존
- ✅ 추가 작업 없음

**단점:**
- 복원 전에 사전 설정 필요

---

### **방법 2: 네트워크 비활성화 상태에서 복원** ⭐⭐

**단계:**
1. 비행기 모드 활성화 (또는 WiFi/모바일 데이터 모두 비활성화)
2. 백업 파일 복원 (동기화할 수 없음)
3. 앱 실행 (AutoPlaylistScreen 로드)
4. 비행기 모드 해제
5. 필요시 수동으로 "새로고침" 버튼 클릭

**동작 원리:**
```
SyncUtils.kt#L426:
if (!isLoggedIn()) {
    Timber.w("Skipping syncLikedSongs - user not logged in")
    return@withContext  // ← 네트워크 없으면 여기서 반환
}
```

**장점:**
- ✅ 동기화 자동 실행 방지
- ✅ liked 리스트 100% 보존

**단점:**
- 복원 중 네트워크 비활성화 필요

---

### **방법 3: 복원 직후 YouTube 로그아웃** ⭐⭐⭐ (매우 실용적)

**단계:**
1. 백업 복원 (동기화 발생)
2. 앱 설정 → 계정 설정 → "로그아웃" 클릭
3. 확인 대화상자에서 "로그아웃" 선택
4. 앱이 자동 동기화 스킵 (로그인되지 않았으므로)
5. 다시 로그인 필요시 수동으로 로그인

**동작 원리:**
```
SyncUtils.kt#L425-427:
private suspend fun executeSyncLikedSongs() {
    if (!isLoggedIn()) {  // ← 로그아웃하면 여기서 종료
        Timber.w("Skipping syncLikedSongs - user not logged in")
        return@withContext
    }
```

**장점:**
- ✅ 동기화 방지
- ✅ liked 리스트 100% 보존
- ✅ 복원 후에도 가능

**단점:**
- 로그아웃 후 다시 로그인 필요

---

### **방법 4: 복원 전 DataStore 백업 파일 수정** ⭐⭐⭐⭐ (고급)

**방법 설명:**
- 백업 파일 내부의 `settings.preferences_pb` 파일을 수정하여 `YtmSyncKey = false`로 설정

**파일 구조:**
```
backup.zip
├── settings.preferences_pb  ← 이 파일 수정
└── song.db
```

**단계:**

#### 4-1. Linux/Mac에서:
```bash
# 1. 백업 파일을 임시 폴더로 복사
mkdir temp_backup
cd temp_backup
unzip ~/Downloads/Metrolist_backup.backup

# 2. settings.preferences_pb 파일 확인
file settings.preferences_pb

# 3. 텍스트 에디터로 열기 (부분 가독성 있음)
strings settings.preferences_pb | grep -i sync

# 4. 프로토콜 버퍼 파일이므로 직접 편집은 어려움
# → 방법 5 사용 권장
```

#### 4-2. Android Studio / ADB 사용:
```bash
# 1. 기존 설정 백업
adb pull /data/data/com.metrolist.music/files/datastore/settings.preferences_pb

# 2. 복원 수행 (자동 동기화 발생)

# 3. 로그아웃으로 복원 (방법 3)
```

**장점:**
- ✅ 복원 전에 설정 미리 변경 가능
- ✅ 자동 동기화 완벽 차단

**단점:**
- ⚠️ 프로토콜 버퍼 파일이라 직접 편집 어려움
- ⚠️ 고급 기술 필요

---

### **방법 5: 복원 후 즉시 앱 강제 종료 + 설정 변경** ⭐⭐⭐

**단계:**
1. 백업 복원 (기기 설정 > 앱)
2. 앱이 자동으로 실행되려면:
   - 앱 아이콘 클릭 하지 말 것
   - 또는 시스템 설정에서 앱 강제 종료
3. ADB 또는 다른 앱으로 데이터 변경
4. 앱 재실행

```bash
# ADB로 YtmSync 비활성화 (고급)
adb shell pm grant com.metrolist.music android.permission.READ_EXTERNAL_STORAGE
adb shell am start -n com.metrolist.music/.ui.screens.settings.AccountSettings
```

**장점:**
- ✅ 동기화 방지

**단점:**
- ⚠️ ADB 설정 필요
- ⚠️ 기술 수준 높음

---

## 🏆 **최종 권장 조합 전략**

### **1차 선택: 방법 1 (복원 전 동기화 비활성화)**
```
복원 전:
1. 설정 → 계정 설정 → "YouTube와 동기화" OFF
2. 백업 생성
3. 데이터 삭제
4. 백업 복원
5. 앱 시작 (자동 동기화 안 함)
```

### **2차 선택: 방법 3 (복원 직후 로그아웃)**
```
복원 후:
1. 백업 복원 (동기화 발생할 수 있음)
2. 설정 → 계정 설정 → "로그아웃"
3. 확인 (로그아웃 완료)
4. 이제 동기화 발생 안 함
5. 필요시 다시 로그인
```

### **3차 선택: 방법 2 (네트워크 비활성화)**
```
복원 중:
1. 비행기 모드 활성화
2. 백업 복원
3. 앱 시작
4. 비행기 모드 해제
5. 필요시 새로고침
```

---

## 🔍 **동기화 상태 확인**

### **YtmSync 설정 확인 (시스템 설정에서):**
```
앱 → 설정 → 계정 설정 → "YouTube와 동기화" 토글 확인
OFF = 동기화 비활성화 ✓
ON = 동기화 활성화 (위험)
```

### **로그인 상태 확인:**
```
앱 → 설정 → 계정 설정 → 상단의 계정 정보
로그인됨 = 동기화 가능
로그아웃 = 동기화 불가능
```

### **로그 확인 (개발자 모드):**
```
개발자 모드 활성화 후:
로그캣에서 "RESTORE" 또는 "syncLikedSongs" 검색
Syncing liked songs 메시지 = 동기화 진행 중
Skipping syncLikedSongs = 동기화 건너뜀
```

---

## ⚠️ **피해야 할 것들**

1. ❌ 복원 후 "새로고침" 버튼 누르기
   - liked 리스트와 YouTube 동기화 시작
   
2. ❌ AutoPlaylistScreen에서 "리스트 새로고침" 버튼 누르기
   - 원격 동기화 발생

3. ❌ 계정 설정에서 "지금 동기화" 버튼 누르기
   - 즉시 동기화 실행

4. ❌ 로그인된 상태에서 복원하기
   - 자동 동기화 발생

---

## 📊 **각 방법별 비교표**

| 방법 | 난이도 | 안전성 | 소요시간 | 추천도 |
|------|--------|--------|---------|--------|
| 1. 복원 전 동기화 비활성화 | ⭐ | ⭐⭐⭐⭐⭐ | 2분 | ⭐⭐⭐⭐⭐ |
| 2. 네트워크 비활성화 | ⭐ | ⭐⭐⭐⭐ | 3분 | ⭐⭐⭐⭐ |
| 3. 복원 후 로그아웃 | ⭐ | ⭐⭐⭐⭐ | 2분 | ⭐⭐⭐⭐⭐ |
| 4. DataStore 파일 수정 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 5분 | ⭐⭐⭐ |
| 5. 강제 종료 + ADB | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 10분 | ⭐⭐ |

---

## 🎯 **즉시 실행 가능한 단계 (방법 3 - 가장 빠름)**

만약 이미 복원했다면:

```
1. 설정(앱 내) 열기
2. 계정 설정 클릭
3. 상단 프로필 이름 클릭
4. "로그아웃" 선택
5. 확인 클릭
6. 완료! liked 리스트가 더 이상 변경되지 않음
```

---

## 📝 **참고: 동기화 코드 위치**

관심 있으신 분들을 위해 코드 위치:

```
SyncUtils.kt (line 425-465)
├─ executeSyncLikedSongs()
│  ├─ YouTube.playlist("LM") 호출 (로그인 필요)
│  └─ 로컬 곡 vs 원격 곡 비교 후 삭제

AutoPlaylistScreen.kt (line 217-222)
├─ LaunchedEffect(Unit)
│  └─ ytmSync=true이면 syncLikedSongs() 자동 호출

PreferenceKeys.kt (line 57)
└─ YtmSyncKey = booleanPreferencesKey("ytmSync")
```

---

## 💡 **그 외 팁**

### **liked 리스트 수동 동기화 (필요시):**
- AutoPlaylistScreen에서 아래로 스와이프
- 또는 새로고침 버튼 클릭
- 이때만 YouTube와 동기화됨

### **선택적 동기화:**
- 설정 → 계정 설정 → "선택 플레이리스트 동기화"
- 특정 플레이리스트만 동기화 가능

### **동기화 로그 확인:**
- 설정 → 개발자 모드 → Logcat에서 확인
