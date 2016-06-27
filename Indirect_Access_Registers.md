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

中间层sequencer上的sequence item类型是uvm_reg_item，也即是寄存器的读或者写操作了，其是class。
在test里，start translation sequence。
```verilog
class my_test extends uvm_test;
  block_env env;
  virtual function void run();
    my_reg_sequence seq = my_reg_sequence::type_id::create(“seq”,this);
    seq.start(env.reg_seqr);
  endfunction
endclass
```
---
## 实际问题
在我的实际应用中，是想把一个寄存器操作，转化为多个寄存器的读和写。这里，我把间接寄存器放在了一个独立的register map上(名为UnAddressedModules)，而不是default map。
```verilog
class wifi_t extends uvm_reg_block;
...
  uvm_reg_map wifi_top;
  uvm_reg_map UnAddressedModules;
...
  virtual function void build();
    wifi_top = create_map("wifi_top", 'h0, 4, UVM_LITTLE_ENDIAN, 1);
    UnAddressedModules = create_map("UnAddressedModules", 'h0, 4, UVM_LITTLE_ENDIAN, 1);
    default_map = wifi_top;
...    
endclass
```

在env中，集成register model以及构建layer sequence。
```verilog
  virtual function void build_phase(uvm_phase phase);
    reg_seqr = uvm_sequencer#(uvm_reg_item)::type_id::create("reg_seqr", this);
    
    rdb = wifi_t::type_id::create("rdb", this);
    rdb.configure(null,"");
    rdb.build();
    rdb.reset();
    rdb.lock_model();
  endfunction
  
  virtual function void connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    
    vsequencer.ahb_master_sqr = ahb_master_uvc.sequencer;
    vsequencer.reg_seqr       = reg_seqr;
    rdb.default_map.set_sequencer(vsequencer.ahb_master_sqr, reg2ahb);
    rdb.default_map.set_auto_predict(1);
    
    rdb.UnAddressedModules.set_sequencer(reg_seqr, null); // 注意没有设置adapter
    rdb.UnAddressedModules.set_auto_predict(1);

    reg_tl_seq = reg_tl_seq_t::type_id::create("reg_tl_seq",,get_full_name());
    req_tl_seq.reg_seqr = reg_seqr; // 设置upstream sequencer
    req_tl_seq.rdb = this.rdb;
    req_tl_seq.adapter = reg2ahb;
    reg_tl_seq.phy_bank_cfg = env_cfg.phy_bank_cfg;
    
  endfunction
    
  virtual task main_phase(uvm_phase phase);
    super.main_phase(phase);
    env_cfg.print();
    reg_tl_seq.start(vsequncer.ahb_master_sqr);
  endtask
```

回过头来，参考uvm_reg_sequence.svh。
```verilog
  virtual task body();
    ...
    forever begin
      uvm_reg_item reg_item;
      reg_seqr.peek(reg_item);
      do_reg_item(reg_item);
      reg_seqr.get(reg_item);
      #0;
    end
  endtask
```

所以，我自己的sequence为下面的代码，因为不是uvm_reg_sequence的子类，所以需要手动实现上面body类似代码。
```verilog
  virtual task body();
    ...
    forever begin
      uvm_reg_item req;
      reg_seqr.peek(req);
      select_bank(0);
      if (req.kind == UVM_READ) begin
        read_data(req);
      end
      else begin
        write_data(req);
      end
      reg_seqr.get(req);
      #0;
      req.end_tr();
    end
```

注意，reg_seqr没有对应的driver与之相连，所以要手动end_tr，否则sequence在发出第一个包后，卡住不动，因为没有consume掉这个transaction。可以参考代码uvm_reg_map.svh。
```verilog
task uvm_reg_map::do_write(uvm_reg_item rw);
  ...
  rw.parent.start_item(rw,rw.prior);
  rw.parent.finish_item(rw);
  rw.end_event.wait_on(); // 等待rw被consume掉
endtask
```
---
## 总结
  层次化的集成register model的方法，处理间接访问寄存器，可行有效，但要注意一些小细节，比如关于transaction生命周期的。
