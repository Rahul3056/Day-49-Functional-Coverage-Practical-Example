Imagine we have a simple arbiter module. This arbiter takes requests from two sources (`req1`, `req2`) and grants access to a shared resource to one of them using grant signals (`gnt1`, `gnt2`). It also has a priority scheme: if both request at the same time, `req1` gets priority.

Here's a basic SystemVerilog model of our arbiter:

```systemverilog
module arbiter (
  input logic req1, req2, clk, rst_n,
  output logic gnt1, gnt2
);

  typedef enum logic [1:0] {IDLE, GNT1, GNT2} state_t;
  state_t current_state, next_state;

  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n)
      current_state <= IDLE;
    else
      current_state <= next_state;
  end

  always_comb begin
    next_state = current_state;
    gnt1 = 0;
    gnt2 = 0;

    case (current_state)
      IDLE: begin
        if (req1 && !req2)
          next_state = GNT1;
        else if (!req1 && req2)
          next_state = GNT2;
        else if (req1 && req2)
          next_state = GNT1; // req1 has priority
        else
          next_state = IDLE;
      end
      GNT1: begin
        gnt1 = 1;
        next_state = IDLE;
      end
      GNT2: begin
        gnt2 = 1;
        next_state = IDLE;
      end
      default: next_state = IDLE;
    endcase
  end

endmodule
```

Now, let's think about what aspects of this arbiter's functionality we want to cover. Some key behaviors we should verify are:

1.  **Granting when only one request is active:**
    * `req1` is high, `req2` is low, and `gnt1` becomes high.
    * `req1` is low, `req2` is high, and `gnt2` becomes high.
2.  **Priority handling:**
    * Both `req1` and `req2` are high, and `gnt1` becomes high (due to priority).
3.  **Idle state:**
    * When no requests are active, the arbiter stays in the `IDLE` state.
4.  **Granting and returning to idle:**
    * After a grant is issued, the arbiter eventually returns to the `IDLE` state.

To cover these functional aspects, we can create a `covergroup` in our testbench:

```systemverilog
class arbiter_coverage;
  covergroup arbiter_cg @(posedge clk);
    // Cover point for single request scenarios
    cp_single_req: coverpoint req1 iff (!req2) {
      bins req1_high = (1);
    }
    cp_single_req: coverpoint req2 iff (!req1) {
      bins req2_high = (1);
    }

    // Cover point for priority scenario
    cp_priority: coverpoint req1 && req2 {
      bins both_high = (1);
    }

    // Cover point for idle state
    cp_idle: coverpoint current_state {
      bins idle_state = IDLE;
    }

    // Cover point for grant transitions
    cp_grant_transition: coverpoint current_state iff (gnt1 || gnt2) {
      bins granting = {GNT1, GNT2};
      bins next_idle = (IDLE);
      transitions (granting => next_idle);
    }
  endgroup : arbiter_cg

  function new();
    arbiter_cg = new();
  endfunction

  task sample_coverage();
    arbiter_cg.sample();
  endtask
endclass : arbiter_coverage
```

Let's break down this `covergroup`:

* **`covergroup arbiter_cg @(posedge clk);`**: This declares a covergroup named `arbiter_cg` that samples its cover points at the positive edge of the clock.
* **`cp_single_req: coverpoint req1 iff (!req2) { ... }`**: This defines a cover point named `cp_single_req`. The `iff (!req2)` condition ensures this cover point is only active when `req2` is low. Inside, we define a bin `req1_high` that gets hit when `req1` is high under this condition. We have a similar cover point for when `req2` is high and `req1` is low.
* **`cp_priority: coverpoint req1 && req2 { ... }`**: This cover point `cp_priority` is active when both `req1` and `req2` are high, and the bin `both_high` tracks this condition.
* **`cp_idle: coverpoint current_state { ... }`**: This cover point tracks the `current_state` of our arbiter, and the `idle_state` bin checks if the state is `IDLE`.
* **`cp_grant_transition: coverpoint current_state iff (gnt1 || gnt2) { ... }`**: This cover point is active when either `gnt1` or `gnt2` is high. We define a bin `granting` that includes the `GNT1` and `GNT2` states and another bin `next_idle` for the `IDLE` state. The `transitions (granting => next_idle)` construct specifically tracks if the arbiter transitions from a granting state back to the `IDLE` state.
* **`function new(); ... endfunction`**: This is the constructor for our coverage class, which instantiates the `arbiter_cg`.
* **`task sample_coverage(); ... endtask`**: This task provides a convenient way to sample the covergroup.

In your testbench, you would instantiate this `arbiter_coverage` class and call the `sample_coverage()` task at appropriate times during your simulation (e.g., after applying stimulus to the arbiter).

```systemverilog
module arbiter_tb;
  logic req1_tb, req2_tb, clk_tb, rst_n_tb;
  logic gnt1_tb, gnt2_tb;

  arbiter dut (
    .req1(req1_tb),
    .req2(req2_tb),
    .clk(clk_tb),
    .rst_n(rst_n_tb),
    .gnt1(gnt1_tb),
    .gnt2(gnt2_tb)
  );

  arbiter_coverage cov;

  initial begin
    clk_tb = 0;
    forever #5 clk_tb = ~clk_tb;
  end

  initial begin
    rst_n_tb = 0;
    #10;
    rst_n_tb = 1;

    cov = new();

    // Apply stimulus to exercise different scenarios
    req1_tb = 1; req2_tb = 0; #20; cov.sample_coverage();
    req1_tb = 0; req2_tb = 1; #20; cov.sample_coverage();
    req1_tb = 1; req2_tb = 1; #20; cov.sample_coverage();
    req1_tb = 0; req2_tb = 0; #10; cov.sample_coverage();
    req1_tb = 1; req2_tb = 0; #15; cov.sample_coverage();
    req1_tb = 0; req2_tb = 0; #10; cov.sample_coverage();
    req1_tb = 0; req2_tb = 1; #15; cov.sample_coverage();
    req1_tb = 0; req2_tb = 0; #10; cov.sample_coverage();
    req1_tb = 1; req2_tb = 1; #15; cov.sample_coverage();
    req1_tb = 0; req2_tb = 0; #20; cov.sample_coverage();

    #100;
    $finish;
  end

endmodule
```

After running the simulation, your simulation tool will report the coverage achieved for each cover point and bin in your `arbiter_cg`. The goal is to reach 100% coverage for all defined bins, indicating that you have thoroughly tested the intended functionality.

This example demonstrates a basic application of functional coverage. In more complex designs, you might have more intricate cover groups with cross coverage (to see the interaction between different signals or state variables) and more sophisticated binning strategies.

Functional coverage is crucial because it helps you move beyond simply checking if the output matches the expected behavior for a few test cases. It ensures that you have explored the design's behavior across its intended operational space.

