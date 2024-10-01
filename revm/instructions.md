다음과 같이 수많은 EVM Opcode들이 구현되어있다. 다 볼 필요 없음


``` rust

pub fn add<H: Host + ?Sized>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::VERYLOW);
    pop_top!(interpreter, op1, op2);
    *op2 = op1.wrapping_add(*op2);
}

pub fn mul<H: Host + ?Sized>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::LOW);
    pop_top!(interpreter, op1, op2);
    *op2 = op1.wrapping_mul(*op2);
}

pub fn sub<H: Host + ?Sized>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::VERYLOW);
    pop_top!(interpreter, op1, op2);
    *op2 = op1.wrapping_sub(*op2);
}

pub fn div<H: Host + ?Sized>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::LOW);
    pop_top!(interpreter, op1, op2);
    if !op2.is_zero() {
        *op2 = op1.wrapping_div(*op2);
    }
}

pub fn sdiv<H: Host + ?Sized>(interpreter: &mut Interpreter, _host: &mut H) {
    gas!(interpreter, gas::LOW);
    pop_top!(interpreter, op1, op2);
    *op2 = i256_div(op1, *op2);
}
```
