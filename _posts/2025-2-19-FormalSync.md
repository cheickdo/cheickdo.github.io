---
layout: post
title: How do I deal with Sequentials in Yosys Formal
---

Previously I wrote about setting up Yosys and Rosette to build your own Formal Verification backend. This was severly limited in scope as it only allowed you to deal with small combinational circuits. This can be extended to small sequential circuits.

A big difference when you're jumping from a combinational circuit to a sequential circuit in the context of SMT solvers, is that now we need to extend into the temporal dimension when we're characterising the input-output relationship of our circuit. This becomes much more complicated when you need to consider multiple clock domains and flip-flop types. So in this exceptionally simple case-study we will ignore this.

When dealing with synchronous circuits, we ask of Yosys to do a variety of passes for us. We are interesting in the *memory_map* and *formalff* commands in particular here. Memory_map starts a pass which implements **$mem_v2** memory cells as DFF's and address decoders [Yosys Documentation](https://yosyshq.readthedocs.io/projects/yosys/en/latest/cell/word_mem.html). This would deal with the declared registers fifo in the sync_fifo module.

```verilog
module sync_fifo #(parameter DEPTH = 4, DATA_WIDTH=32) (
  input clk, resetn,
  input enq, deq,
  input [DATA_WIDTH-1:0] data_in,
  output reg [DATA_WIDTH-1:0] data_out,
  output full, empty
);
  
reg [$clog2(DEPTH)-1:0] w_ptr, r_ptr;
reg [DATA_WIDTH-1:0] fifo [DEPTH];
reg [$clog2(DEPTH)-1:0] count;
...
endmodule
```

```bash
-- Running command `memory_map' --

3. Executing MEMORY_MAP pass (converting memories to logic and flip-flops).
Mapping memory \fifo in module \sync_fifo:
  created 4 $dff cells and 0 static cells of width 32.
  read interface: 0 $dff and 3 $mux cells.
  write interface: 4 write mux blocks.
```

By running the formalff -clk2ff pass, we can prepare our flip-flops for formal verification. This takes all the clocked flip-flops (assuming a global clock), and replaces them with $ff cells, essentially removing dependence of the design to the clock (negative hold-time is assumed). 

```racket
(struct
  sync_fifo_State
  ($procdff$61
    $memory_fifo_0_$68
    $memory_fifo_1_$70
    $memory_fifo_2_$72
    $memory_fifo_3_$74
    $procdff$60
    $procdff$65
    $procdff$59)
  #:transparent
  ; $procdff$61 (bitvector 2)
  ; $memory_fifo_0_$68 (bitvector 32)
  ; $memory_fifo_1_$70 (bitvector 32)
  ; $memory_fifo_2_$72 (bitvector 32)
  ; $memory_fifo_3_$74 (bitvector 32)
  ; $procdff$60 (bitvector 2)
  ; $procdff$65 (bitvector 2)
  ; $procdff$59 (bitvector 32)
)
```

The idea then is that we have unrolled our state into input state and output state. It is important to also consider that the point with which our state is sliced is taken to be at the *input* to the flip-flops. 

This means that simply taking the *essentially* combinational output of our unrolled model given some input and input state will give us the output corresponding to the *previous* state and not the *current* state. Note our output in the counter and sync_fifo cases are registered so inputs would take 2 passes to propogate.

In our counter case, this can be more simply interpreted as the $0_count_3_0_ value holding the previous count value will propogate to to the present, while this value will be updated in the output state to hold our newly computed count value.

```racket
(counter_State
	$0_count_3_0_ ; $procdff$10
)
```

```verilog
//counter
module counter(
    input clk, resetn,
    input en,
    output reg [3:0] count
);

always@(posedge clk) begin
    if(!resetn) begin
        count <= 0;
    end
    else if (en) begin
            count <= count_d;
    end
end

always @(*)
    count_d = count + 1;

endmodule
```

Looking back at our very simple counter case, given a specification that our counter will not reset during operation and it must always increase we can check this satisfaction in the following function.


```racket
(define (check-sat input state)
    (define out0 (car (counter input state)))
    (define out0_state (cdr (counter input state)))
    (define data_out0 (counter_Outputs-count out0) )

    (define out1 (car (counter input out0_state)))
    (define data_out1 (counter_Outputs-count out1) )

    (define resetn (counter_Inputs-resetn input))

    (assume (bveq resetn (bv 1 1)))
    (assert (bvuge data_out1 data_out0))
    ; (assert (bvule data_out1 (bv 0 4)))

)
```

Where 2 passes allows us to propogate through our input and states to see an overflow case.

```racket
(model
 [s_resetn (bv #b1 1)]
 [s_en (bv #b1 1)]
 [$procdff$9 (bv #x4 4)])
```

