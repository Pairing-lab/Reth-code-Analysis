Revm에 대한 Structure 분석 자료
거추장한 것만 빼고 보면, evm에는 state와 관련이 깊은 Context와 수행 로직을 담당하는 Handler가 있다. 
Handler는 Opcode 실행 전 및 실행 후, 실행 시 어떻게 실행해야하는지 등을 추상화된 레벨에서 기술하고 있다. 
하지만 자세한 Opcode 동작 및 실행 시 어떻게 되는지 등은 Interpreter에서 디렉토리에서 찾을 수 있다.

Primtitives와 Precompile은 일종의 `building block`이라고 보면 될 것.
따라서, revm(evm) 디렉토리와 Interpreter를 보면 될 것이다 .

[evm](./evm.md)

[Interpreter](./Interpreter.md)

[Primitives](./Primitives.md)

[Precompile](./Precompile.md)


![alt text](image.png)


