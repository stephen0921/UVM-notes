# get_next_item会block sequencer，不去调度其他的sequence
```verilog
task seq::body()
  start_item(item);
  item.randomize();
  finish_item(item);
  
endtask

task driver::run_phase();
  forever begin
    @(posedge vif.clk);
    seq_item_port.get_next_item(req);
    driver_transfer(req);
    seq_item_port.item_done();
  end
endtask
```

# get不会block sequencer，为多线程的处理提供了支持
```verilog
task seq::body()
  start_item(item);
  item.randomize();
  finish_item(item);
  get_response(rsp);
endtask

task driver::run_phase();
  forever begin
    @(posedge vif.clk);
    seq_item_port.get(req);
    driver_transfer(req);

    seq_item_port.put(rsp);
  end
endtask
```
