# JDK 25 FFM API (Foreign Function & Memory)

> 한 줄 요약: JDK 25에서 정식화된 FFM API를 학습한다. JNI/ByteBuffer 대체, 네이티브 메모리·함수 호출을 100% Java로 안전·고성능으로 다루는 방법.

## 배경
유튜브 영상 "Native Interoperability with JDK 25 and the FFM API" (https://www.youtube.com/watch?v=_4bEHwJ4Bdk) 를 시청하고 정리·심화 학습.

영상 요약 키 포인트:
- **FFM API의 장점**: JNI처럼 C 코드/shim 라이브러리 없이 100% Java로 네이티브 통합. (0:00–1:23)
- **Memory API**: `MemorySegment` 로 힙·네이티브 메모리를 통일된 추상화로 관리, `Arena` 로 생명주기를 결정론적으로 제어. (9:21–13:43)
- **Memory Layout & VarHandle**: C 구조체에 대응하는 `MemoryLayout`, 안전·효율적인 접근을 위한 `VarHandle`. (16:02–23:37)
- **Function API**: `Linker` 로 네이티브 함수 직접 호출. (40:15–43:30)
- **jextract**: C 헤더에서 Java 바인딩 자동 생성. (45:21–46:04)
- **활용 예**: ONNX Runtime 같은 고성능 네이티브 라이브러리를 Java에서 직접 사용.

---

## 관련 자료
- 영상: https://www.youtube.com/watch?v=_4bEHwJ4Bdk
- JEP 454 (Foreign Function & Memory API, JDK 22 finalized → 25 유지)
- jextract: https://github.com/openjdk/jextract
