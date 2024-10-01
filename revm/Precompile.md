Precompile은 EVM에서 직접 구현된 함수를 실행하기 위한 일종의 `편법`이다.
일반적인 Smart Contract와 다르게, 이미 EVM 자체에 내장되어 있는 기능들이다.
비유를 들자면 Lookup 테이블 처럼 생각할 수 있을 것이다. 
Precompile은 계산적으로 복잡하고 무거운 연산을 더 효율적으로 처리하기 위해 사용된다. 이러한 연산들을 일반 스마트 컨트랙트로 구현하면 비용이 많이 들기 때문에, `미리` 정의된다.

***기억할 것***
- 하드코딩된 주소에 위치
- 실제 컨트랙트가 아닌, EVM에 내장된 기능
- 일반적인 Smart Contract다 실행 비용이 낮음

REVM에서는 6가지가 구현되어있다.

1) blake2: BLAKE2 해시 함수를 위한 precompile
2) bn128 curve: BN128 타원 곡선 연산을 위한 precompile
3) identity: 입력을 그대로 출력하는 간단한 precompile
4) secp256k1: secp256k1 타원 곡선 연산을 위한 precompile
5) modexp: 모듈러 지수 연산을 위한 precompile
6) sha256 및 ripemd160: 각각 SHA-256과 RIPEMD-160 해시 함수를 위한 precompile


결론: 자세히 볼 필요는 없는 Crate