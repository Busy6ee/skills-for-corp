# XiangShan

- **Source URL**: https://github.com/OpenXiangShan/XiangShan
- **License**: Mulan PSL v2
- **Import Date**: 2026-03-22
- **Branch Reference**: kunminghu-v3 (Kunminghu V3 generation)
- **Chisel Version**: 7.3.0 (Scala 2.13.17)
- **Imported Skills**:
  - `xiangshan-architecture`: 레포 구조, 모듈 계층, 신호 흐름
  - `xiangshan-parameters`: XSCoreParameters 시스템, DSE 파라미터
  - `xiangshan-conventions`: 네이밍, 파이프라인 패턴, Bundle/IO 관례
  - `xiangshan-build`: Mill 빌드, Makefile 시뮬레이션, 환경 설정
  - `xiangshan-debug`: difftest, 성능 카운터, ChiselDB, 로그 분석
  - `xiangshan-frontend`: BPU, FTQ, IFU, ICache, IBuffer
  - `xiangshan-backend`: Decode, Rename, Dispatch, IQ, ROB, DataPath
  - `xiangshan-memory`: MemBlock, LSQ, DCache, MMU, 프리페치
- **Modifications**: XiangShan 레포 소스 코드 및 docs에서 아키텍처 핵심 추출, AI 코드 생성 특화 gotcha 추가
- **Excluded Content**: 전체 RTL 소스, 테스트벤치, CI/CD 설정, 서드파티 서브모듈 소스
