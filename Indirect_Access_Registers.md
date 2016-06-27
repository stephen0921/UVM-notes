## 序

UVM register model为我们的寄存器验证提供了很大的帮助，多种集成register model的方法，尤其是使用layered register sequencer的方法，方便了间接访问寄存器的验证。

---

## uvm_users_guide_1.1的例子分析

参考第5.9.2.3小节，

```verilog
// translation sequence type
typedef uvm_reg_sequence #(uvm_sequence #(apb_rw)) reg2apb_seq_t;

class block_env extends uvm_env;
  block_reg_model regmodel;
  uvm_sequencer#(uvm_reg_item) reg_seqr;
  apb_agent apb;
  reg2apb_seq_t reg2apb_seq;
  
  virtual function void connect();
    if (regmodel.get_parent() == null) begin
      regmodel.default_map.set_sequencer(reg_seqr,null);
      reg2apb_seq = reg2apb_seq_t::type_id::create(“reg2apb_seq”,,
      get_full_name());
      reg2apb_seq.reg_seqr = reg_seqr;
      reg2apb_seq.adapter =
      reg2apb_adapter::type_id::create(“reg2apb”,,
      get_full_name());
      regmodel.set_auto_predict(1);
    end
  endfunction

  virtual task run();
    reg2apb_seq.start(apb.sequencer);
  endtask
endclass
```