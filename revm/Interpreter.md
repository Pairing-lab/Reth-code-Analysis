인터프리터 크레이트는 EVM Opcode의 실행을 담당하며, Opcode를 단계별로 실행하는 Event Loop 역할을 하ㄴ다. 

Module:
[gas](./gas.md): EVM에서의 가스 메커니즘을 처리하며, 예를 들어 작업에 대한 가스 비용을 계산한다.
[host](./host.md): EVM 컨텍스트 Host 트레이트를 정의한다.
[interpreter_action](interpreter_action.md): EVM 구현에 사용되는 데이터 구조를 포함한다.
[instruction_result](instruction_result.md): 명령어 실행 결과를 정의한다.
[instructions](instructions.md): EVM Opcode를 정의한다.

