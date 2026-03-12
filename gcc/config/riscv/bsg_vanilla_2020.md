;; Generic DFA-based pipeline description for HammerBlade Superscalar.
;; Dual-issue: Integer/Memory bucket and FP Compute bucket.

(define_automaton "bsg_vanilla_2020")

;; Define the two main pipes for superscalar issue
(define_cpu_unit "bsg_vanilla_2020_int_pipe" "bsg_vanilla_2020")
(define_cpu_unit "bsg_vanilla_2020_fp_pipe" "bsg_vanilla_2020")

;; Specialized units for long-latency operations
(define_cpu_unit "bsg_vanilla_2020_idiv" "bsg_vanilla_2020")
(define_cpu_unit "bsg_vanilla_2020_fdiv" "bsg_vanilla_2020")

;; ============================================================
;; INTEGER BUCKET (Includes Integer ALU, Loads, Stores, and FP Loads/Stores)
;; ============================================================

;; Standard Integer ALU
(define_insn_reservation "bsg_vanilla_2020_int" 1
  (and (eq_attr "tune" "bsg_vanilla_2020")
       (eq_attr "type" "unknown,const,arith,shift,slt,multi,auipc,nop,logical,move"))
  "bsg_vanilla_2020_int_pipe")

;; Remote load (still uses integer pipe for address generation)
(define_insn_reservation "bsg_vanilla_2020_remote_load" 32
  (and (eq_attr "tune" "bsg_vanilla_2020")
       (eq_attr "type" "load,fpload")
       (eq_attr "remote_mem_op" "yes"))
  "bsg_vanilla_2020_int_pipe")

;; Integer local load
(define_insn_reservation "bsg_vanilla_2020_load" 2
  (and (eq_attr "tune" "bsg_vanilla_2020")
       (eq_attr "type" "load")
       (eq_attr "remote_mem_op" "no"))
  "bsg_vanilla_2020_int_pipe")

;; FP local load - TREATED AS INTEGER
;; This allows pairing with an FPU compute instruction.
(define_insn_reservation "bsg_vanilla_2020_fpload" 3
  (and (eq_attr "tune" "bsg_vanilla_2020")
       (eq_attr "type" "fpload")
       (eq_attr "remote_mem_op" "no"))
  "bsg_vanilla_2020_int_pipe")

;; Store (Integer and FP)
(define_insn_reservation "bsg_vanilla_2020_store" 1
  (and (eq_attr "tune" "bsg_vanilla_2020")
       (eq_attr "type" "store,fpstore"))
  "bsg_vanilla_2020_int_pipe")

;; Integer Multiplication
(define_insn_reservation "bsg_vanilla_2020_imul" 2
  (and (eq_attr "tune" "bsg_vanilla_2020")
       (eq_attr "type" "imul"))
  "bsg_vanilla_2020_int_pipe")

;; Integer Division
(define_insn_reservation "bsg_vanilla_2020_idiv" 34
  (and (eq_attr "tune" "bsg_vanilla_2020")
       (eq_attr "type" "idiv"))
  "bsg_vanilla_2020_int_pipe,bsg_vanilla_2020_idiv*33")

;; ============================================================
;; FLOATING POINT BUCKET (FPU Compute)
;; ============================================================

;; Standard FPU arithmetic
(define_insn_reservation "bsg_vanilla_2020_fpu_float" 3
  (and (eq_attr "tune" "bsg_vanilla_2020")
       (eq_attr "type" "fadd,fmul,fmadd"))
  "bsg_vanilla_2020_fp_pipe")

;; FP Compare
(define_insn_reservation "bsg_vanilla_2020_fcmp" 1
  (and (eq_attr "tune" "bsg_vanilla_2020")
       (eq_attr "type" "fcmp"))
  "bsg_vanilla_2020_fp_pipe")

;; FP Move
(define_insn_reservation "bsg_vanilla_2020_fmove" 3
  (and (eq_attr "tune" "bsg_vanilla_2020")
       (eq_attr "type" "fmove"))
  "bsg_vanilla_2020_fp_pipe")

;; FP Division / Square Root
(define_insn_reservation "bsg_vanilla_2020_fdiv" 26
  (and (eq_attr "tune" "bsg_vanilla_2020")
       (eq_attr "type" "fdiv,fsqrt"))
  "bsg_vanilla_2020_fp_pipe,bsg_vanilla_2020_fdiv*25")

;; ============================================================
;; CROSS-PIPE & SPECIAL CASES
;; ============================================================

;; i2f / f2i (Moves between register files)
;; These are assigned to the integer pipe as they typically sync or stall.
(define_insn_reservation "bsg_vanilla_2020_i2f" 3
  (and (eq_attr "tune" "bsg_vanilla_2020")
       (eq_attr "type" "mtc"))
  "bsg_vanilla_2020_int_pipe")

(define_insn_reservation "bsg_vanilla_2020_f2i" 1
  (and (eq_attr "tune" "bsg_vanilla_2020")
       (eq_attr "type" "mfc"))
  "bsg_vanilla_2020_int_pipe")

;; Branch restriction: "An instruction after a conditional branch cannot be paired."
;; We model this by making the branch consume BOTH issue slots.
(define_insn_reservation "bsg_vanilla_2020_branch" 1
  (and (eq_attr "tune" "bsg_vanilla_2020")
       (eq_attr "type" "branch,jump,call"))
  "bsg_vanilla_2020_int_pipe + bsg_vanilla_2020_fp_pipe")
