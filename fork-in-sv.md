# fork in system verilog

## fork join

所有child process结束，parent process才向下进行

## fork join any

任一个child process结束，parent process向下进行

## fork join none

parent process不等待child process结束，就向下进行

```verilog
function void my_component::phase_ready_to_end(uvm_phase phase);
  if(!is_ok_to_end()) begin
    phase.raise_objection(this, "not yet ready to end phase");
    fork begin
      wait_for_ok_end();
      phase.drop_objection(this, "ok to end phase");
    end
    join_none
  end
endfunction: phase_ready_to_end
```
根据实际情况，来实现is_ok_to_end和wait_for_ok_end，并不是uvm的class函数。
