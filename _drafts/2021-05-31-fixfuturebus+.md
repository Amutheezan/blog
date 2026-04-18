---
type: posts
title: Simple Fix on Futurebus+ Cache Coherence Protocol
author: Amutheezan Sivagnanam
category: Computer Architecture
date: 2021-05-31
tags:
- architecture
- futurebus+
- murphi

---

I have implemented a simple fixing for Futurebus+ cache coherence protocol which I explained in
my [previous post](https://amutheezan.com/computer%20architecture/futurebus+/).

### Initial Setup & Steps

Initially clone the [repository](https://github.com/Amutheezan/futurebus) and go to the code ```main``` directory and
execute the following two command to start the docker instances (make sure docker is already installed in your computer
and started; check here for installation steps of [docker](https://docs.docker.com/get-docker/).

```docker-compose build```

```docker-compose run ubuntu bash```

**Note**: the docker implementation and murphi3.1 integration obtained from the
following [repository](https://github.com/adnaneGdihi/fixed_murphi3.1).

After that you will get into the bash follow the steps shown below in the ubuntu bash,

* First go to the directory ```Murphi3.1/src``` and compile the Murphi using ```make``` command (if it not already
  generated). And optionally make the executable access using the command ```chmod +x Murphi3.1/src/mu```, if you in the
  home directory.
* Then go to the directory ```verification``` and obtain the C file for the Futurebus+ verification
  using ```./../Murphi3.1/src/mu futurebus.m``` command (if it not already generated). This will generate the C
  file ```futurebus.C```.
* Thereafter compile the generate file using the command ```make futurebus```, this eventually generate the executable
  program.
* Finally run the executable using the following command ```./futurebus```

### OR

you can perform these entire steps by simply go to the ```verification``` directory and run the command ```sh run.sh```.

## Verification using Murphi3.1

Please note that Murphi3.1 come installed with docker image so no need to worry about downloading and setting up.

I followed the same code structure as previous post.

* constants - to define constants such as number of processor, number of values

```c
const
	processor_count: 3;
	value_count: 1;
```

* define types such as processor states (```ProcState```) using enumeration which represents all the states in
  Futurebus+ protocol, message types (```MessageType```) as enumeration which includes different message used to send
  between the states either bus or cpu call, finally define the message type (```Message```) .

```c
type
  	Proc : scalarset(processor_count);
	Value: scalarset(value_count);
	ProcState:
	record
		state: enum { 
			FB_I,
			FB_EU,
			FB_EM,
			FB_SU,
			FB_PR,
			FB_PEMR,
			FB_PSU,
			FB_PW,
			FB_PEMW
		};
		value: Value;
	end;

	MessageType: enum {  
		ReadShared,
		ReadModified,
		Invalidate
	}; 

	Message:
    record
      mtype: MessageType;
	  value: Value;
    end;
```

* variables - the global variables represents the system.

I have added ```pending_write``` variable to identify whether there are multiple processor with pending writes and avoid
to processor to reach exclusive state due to error in Futurebus+ cache coherence protocol.

```c
var
	proc_state: array[Proc] of ProcState;
	transaction_flag: boolean;
	last_write: Value;
	one_flag: boolean;
	more_flag: boolean;
	pending_write: boolean;
	send_msg: Message;
```

* procedures - contains the function used by the verification. I have update the functions to make sure the processor to
  update the variable ```pending_write```.

```c
procedure SendMessage(msg: Message; i: Proc);
begin
	switch msg.mtype
		case ReadShared:
			if (proc_state[i].state = FB_EM | 
				proc_state[i].state = FB_EU) then
				proc_state[i].state :=  FB_SU;
			endif;

			if (proc_state[i].state = FB_PW) then
				proc_state[i].state :=  FB_PR;
			endif;

		case ReadModified:
			if (proc_state[i].state = FB_SU | 
				proc_state[i].state = FB_PR | 
				proc_state[i].state = FB_PSU | 
				proc_state[i].state = FB_PEMR | 
				proc_state[i].state = FB_EU |
				proc_state[i].state = FB_EM) then
				proc_state[i].state :=  FB_I;
			endif;

			if (proc_state[i].state = FB_PW) then
				proc_state[i].state :=  FB_PR;
			endif;

		case Invalidate:
			proc_state[i].state :=  FB_I;
	endswitch;

    proc_state[i].value := msg.value;
	if (proc_state[i].state = FB_I) then
		proc_state[i].value := undefined;
	endif;
end;

procedure UpdateSignals();
begin

	for m: Proc do
		if proc_state[m].state = FB_PW then
			pending_write := true;
		endif;
	endfor;

	for m: Proc do
		if proc_state[m].state = FB_PEMR | 
		   proc_state[m].state = FB_PSU |
		   proc_state[m].state = FB_EU |
		   proc_state[m].state = FB_EM |
		   proc_state[m].state = FB_SU then
			transaction_flag := true;
		endif;
	endfor;

	for m: Proc do
		if proc_state[m].state = FB_PR then
			if one_flag = true then
				more_flag := true;
			endif;

			if one_flag = false & transaction_flag = false then
				one_flag := true;
			endif;
		endif;
	endfor;
end;
```

* ruleset - define the set of rules that can be used to model the systems. Update the ruleset to avoid two processor
  reach the exclusive state.

```c

ruleset i: Proc do
  	alias p: proc_state[i] do
  		ruleset v: Value do
			rule "Write data"
				(p.state = FB_SU | p.state = FB_EU | p.state = FB_EM)
			==>
				p.state :=  FB_EM;
				p.value := v;
				last_write := v;

				UpdateSignals();

				for k:Proc do
					if i != k then
						send_msg.mtype := Invalidate;
						send_msg.value := undefined;
						SendMessage(send_msg, k);
					endif;
				endfor;

				UpdateSignals();
			endrule;

			rule "Trying to write data"
				(p.state = FB_EM | p.state = FB_PEMW)
			==>
				p.state :=  FB_PEMW;
				p.value := v;

				UpdateSignals();
			endrule;

			rule "Invalidate, on DACKemw"
				(p.state = FB_PEMW)
			==>
				p.state :=  FB_I;
				p.value := undefined;

				UpdateSignals();
			endrule;

			rule "Write data, on DACK" 
				(p.state = FB_PW) 
			==>
				p.state :=  FB_EM;
				p.value := v;
				last_write := v;
				
				UpdateSignals();

				for k: Proc do
					if i != k then
						send_msg.mtype := ReadModified;
						send_msg.value := v;
						SendMessage(send_msg, k);
					endif;
				endfor;

				UpdateSignals();
			endrule;
		
			rule "Write data, on DACKemw" 
				(p.state = FB_PW)
			==>
				p.state :=  FB_EM;
				p.value := v;
				last_write := v;
				
				UpdateSignals();

				for k: Proc do
					if i != k then
						send_msg.mtype := ReadModified;
						send_msg.value := v;
						SendMessage(send_msg, k);
					endif;
				endfor;

				UpdateSignals();
			endrule;
		
			rule "Trying to write data"
				(p.state = FB_I)
			==>
				p.state :=  FB_PW;

				UpdateSignals();
			endrule;

			rule "Trying to write data (pending)"
				(p.state = FB_PW & one_flag = false)
			==>
				p.state :=  FB_PW;
			endrule;
		endruleset;

		rule "Trying to read data"
			(p.state = FB_I)
		==>
			p.state := FB_PR;
							
			UpdateSignals();
		endrule;
			
		rule "Trying to read data (pending)"
			(p.state = FB_PR)
		==>
			p.state := FB_PR;
		endrule;

		rule "Moved to shared state from EMR on DACKem"
			(p.state = FB_PEMR)
		==>
			p.state :=  FB_SU;
			p.value := last_write;

			UpdateSignals();

			for k: Proc do
				if i != k then
					send_msg.mtype := ReadShared;
					send_msg.value := last_write;
					SendMessage(send_msg, k);
				endif;
			endfor;

			UpdateSignals();
		endrule;

		rule "Read shared data"
			(p.state = FB_SU)
		==>
			p.state :=  FB_SU;
			p.value := last_write;

			UpdateSignals();
		endrule;

		rule "Read data from EM"
			(p.state = FB_EM | p.state = FB_PEMR)
		==>
			p.state :=  FB_PEMR;

			UpdateSignals();
		endrule;

		rule "Go to Pending Shared state"
			(p.state = FB_EU | p.state = FB_SU | p.state = FB_PSU)
		==>
			p.state :=  FB_PSU;

			UpdateSignals();
		endrule;


		rule "Go to Shared state on DACK"
			(p.state = FB_PSU)
		==>
			p.state :=  FB_SU;
			p.value := last_write;

			UpdateSignals();

			for k: Proc do
				if i != k then
					send_msg.mtype := ReadShared;
					send_msg.value := last_write;
					SendMessage(send_msg, k);
				endif;
			endfor;

			UpdateSignals();
		endrule;

		rule "Go to Shared state on DACK"
			(p.state = FB_PR & transaction_flag)
		==>
			p.state :=  FB_SU;
			p.value := last_write;

			UpdateSignals();

			for k: Proc do
				if i != k then 
					send_msg.mtype := ReadShared;
					send_msg.value := last_write;
					SendMessage(send_msg, k);
				endif;
			endfor;

			UpdateSignals();
		endrule;

		rule "Go to Shared state on DACK"
			(p.state = FB_PR & transaction_flag = false & more_flag = true)
		==>
			p.state :=  FB_SU;
			p.value := last_write;

			UpdateSignals();

			for k: Proc do
				if i != k then 
					send_msg.mtype := ReadShared;
					send_msg.value := last_write;
					SendMessage(send_msg, k);
				endif;
			endfor;

			UpdateSignals();
		endrule;

		rule "Go to Shared state on DACKem"
			(p.state = FB_PR & transaction_flag = false & more_flag = true)
		==>
			p.state :=  FB_SU;
			p.value := last_write;

			UpdateSignals();

			for k: Proc do
				if i != k then 
					send_msg.mtype := ReadShared;
					send_msg.value := last_write;
					SendMessage(send_msg, k);
				endif;
			endfor;

			UpdateSignals();
		endrule;

		rule "Go to Exclusive state on DACK"
			(p.state = FB_PR & transaction_flag = false & one_flag = true)
		==>
			p.state :=  FB_EU;
			p.value := last_write;

			UpdateSignals();

			for k: Proc do
				if i != k then 
					send_msg.mtype := ReadShared;
					send_msg.value := last_write;
					SendMessage(send_msg, k);
				endif;
			endfor;

			UpdateSignals();
		endrule;

		rule "Read data from Exclusive states"
			(p.state = FB_EM | p.state = FB_EU)
		==>
			p.value := last_write;
			UpdateSignals();
		endrule;
  	endalias;
endruleset;

```

* start-state - define the start state of the system.

```c
startstate
	for i: Proc do
		proc_state[i].state := FB_I;
	endfor;
  transaction_flag := false;
  one_flag := false;
  more_flag := false;
  undefine last_write;
  undefine send_msg;
endstartstate;
```

* invariants - define the cases which determine the correct states of the system, or checking the validity of the
  system. We don't need to change the invariants from initial verification, we can still use the same invariants, and
  check whether any of these invariants failed or not.

```c

Invariant "only one processor in EM state"
ForAll p1 : Proc Do
	ForAll p2 : Proc Do
		proc_state[p1].state = FB_EM & proc_state[p2].state = FB_EM
		->
			p1 = p2
	End
	End;

Invariant "only one processor in EU state"
ForAll p1 : Proc Do
	ForAll p2 : Proc Do
		proc_state[p1].state = FB_EU & proc_state[p2].state = FB_EU
		->
			p1 = p2
	End
	End;

invariant "values in valid state match last write"
  ForAll n : Proc Do  
    proc_state[n].state = FB_SU | proc_state[n].state = FB_EM | proc_state[n].state = FB_EU
    ->
    proc_state[n].value = last_write 
  End;

invariant "value is undefined while invalid"
  ForAll n : Proc Do  
    proc_state[n].state = FB_I
    ->
    IsUndefined(proc_state[n].value)
  End;
  
```

Output of verification of Futurebus+ cache coherence protocol, this provides a brief summary whether any errors or
issues found and other statistics. And we observe no outputs.

```
Protocol: futurebus_fix

Algorithm:
	Verification by breadth first search.
	with symmetry algorithm 3 -- Heuristic Small Memory Normalization
	with permutation trial limit 10.

Memory usage:

	* The size of each state is 104 bits (rounded up to 16 bytes).
	* The memory allocated for the hash table and state queue is
	  8 Mbytes.
	  With two words of overhead per state, the maximum size of
	  the state space is 392159 states.
	   * Use option "-k" or "-m" to increase this, if necessary.
	* Capacity in queue for breadth-first search: 39215 states.
	   * Change the constant gPercentActiveStates in mu_prolog.inc
	     to increase this, if necessary.

Warning: No trace will not be printed in the case of protocol errors!
         Check the options if you want to have error traces.

Status:

	No error found.

State Space Explored:

	476 states, 3233 rules fired in 0.10s.

Analysis of State Space:

	There are rules that are never fired.
	If you are running with symmetry, this may be why.  Otherwise,
	please run this program with "-pr" for the rules information.

```

Next I provides the output of error traces of Futurebus+ traces of cache coherence protocol using the command
suffix ```-tv```. This provides the the reasons why the error happen.l And you observe that now the error doesn't
occurs.

```
Protocol: futurebus_fix

Algorithm:
	Verification by breadth first search.
	with symmetry algorithm 3 -- Heuristic Small Memory Normalization
	with permutation trial limit 10.

Memory usage:

	* The size of each state is 104 bits (rounded up to 16 bytes).
	* The memory allocated for the hash table and state queue is
	  8 Mbytes.
	  With two words of overhead per state, the maximum size of
	  the state space is 392159 states.
	   * Use option "-k" or "-m" to increase this, if necessary.
	* Capacity in queue for breadth-first search: 39215 states.
	   * Change the constant gPercentActiveStates in mu_prolog.inc
	     to increase this, if necessary.

Status:

	No error found.

State Space Explored:

	476 states, 3233 rules fired in 0.10s.

Analysis of State Space:

	There are rules that are never fired.
	If you are running with symmetry, this may be why.  Otherwise,
	please run this program with "-pr" for the rules information.

```

Finally, to observe what are rule fired, I obtain the output of rule firing of Futurebus+ traces of cache coherence
protocol using the command suffix ```-pr```.

```
Protocol: futurebus_fix

Algorithm:
	Verification by breadth first search.
	with symmetry algorithm 3 -- Heuristic Small Memory Normalization
	with permutation trial limit 10.

Memory usage:

	* The size of each state is 104 bits (rounded up to 16 bytes).
	* The memory allocated for the hash table and state queue is
	  8 Mbytes.
	  With two words of overhead per state, the maximum size of
	  the state space is 392159 states.
	   * Use option "-k" or "-m" to increase this, if necessary.
	* Capacity in queue for breadth-first search: 39215 states.
	   * Change the constant gPercentActiveStates in mu_prolog.inc
	     to increase this, if necessary.

Warning: No trace will not be printed in the case of protocol errors!
         Check the options if you want to have error traces.

Status:

	No error found.

State Space Explored:

	476 states, 3233 rules fired in 0.10s.

Rules Information:

	Fired 35 times	- Rule "Read data from Exclusive states, i:Proc_1"
	Fired 19 times	- Rule "Read data from Exclusive states, i:Proc_2"
	Fired 9 times	- Rule "Read data from Exclusive states, i:Proc_3"
	Fired 4 times	- Rule "Go to Exclusive state on DACK, i:Proc_1"
	Fired 5 times	- Rule "Go to Exclusive state on DACK, i:Proc_2"
	Fired 3 times	- Rule "Go to Exclusive state on DACK, i:Proc_3"
	Fired 3 times	- Rule "Go to Shared state on DACKem, i:Proc_1"
	Fired 4 times	- Rule "Go to Shared state on DACKem, i:Proc_2"
	Fired 2 times	- Rule "Go to Shared state on DACKem, i:Proc_3"
	Fired 3 times	- Rule "Go to Shared state on DACK, i:Proc_1"
	Fired 4 times	- Rule "Go to Shared state on DACK, i:Proc_2"
	Fired 2 times	- Rule "Go to Shared state on DACK, i:Proc_3"
	Fired 97 times	- Rule "Go to Shared state on DACK, i:Proc_1"
	Fired 141 times	- Rule "Go to Shared state on DACK, i:Proc_2"
	Fired 72 times	- Rule "Go to Shared state on DACK, i:Proc_3"
	Fired 21 times	- Rule "Go to Shared state on DACK, i:Proc_1"
	Fired 61 times	- Rule "Go to Shared state on DACK, i:Proc_2"
	Fired 81 times	- Rule "Go to Shared state on DACK, i:Proc_3"
	Fired 116 times	- Rule "Go to Pending Shared state, i:Proc_1"
	Fired 120 times	- Rule "Go to Pending Shared state, i:Proc_2"
	Fired 98 times	- Rule "Go to Pending Shared state, i:Proc_3"
	Fired 50 times	- Rule "Read data from EM, i:Proc_1"
	Fired 51 times	- Rule "Read data from EM, i:Proc_2"
	Fired 44 times	- Rule "Read data from EM, i:Proc_3"
	Fired 91 times	- Rule "Read shared data, i:Proc_1"
	Fired 56 times	- Rule "Read shared data, i:Proc_2"
	Fired 16 times	- Rule "Read shared data, i:Proc_3"
	Fired 19 times	- Rule "Moved to shared state from EMR on DACKem, i:Proc_1"
	Fired 35 times	- Rule "Moved to shared state from EMR on DACKem, i:Proc_2"
	Fired 36 times	- Rule "Moved to shared state from EMR on DACKem, i:Proc_3"
	Fired 101 times	- Rule "Trying to read data (pending), i:Proc_1"
	Fired 146 times	- Rule "Trying to read data (pending), i:Proc_2"
	Fired 75 times	- Rule "Trying to read data (pending), i:Proc_3"
	Fired 190 times	- Rule "Trying to read data, i:Proc_1"
	Fired 61 times	- Rule "Trying to read data, i:Proc_2"
	Fired 9 times	- Rule "Trying to read data, i:Proc_3"
	Fired 6 times	- Rule "Trying to write data (pending), v:Value_1, i:Proc_1"
	Fired 26 times	- Rule "Trying to write data (pending), v:Value_1, i:Proc_2"
	Fired 46 times	- Rule "Trying to write data (pending), v:Value_1, i:Proc_3"
	Fired 190 times	- Rule "Trying to write data, v:Value_1, i:Proc_1"
	Fired 61 times	- Rule "Trying to write data, v:Value_1, i:Proc_2"
	Fired 9 times	- Rule "Trying to write data, v:Value_1, i:Proc_3"
	Fired 16 times	- Rule "Write data, on DACKemw, v:Value_1, i:Proc_1"
	Fired 73 times	- Rule "Write data, on DACKemw, v:Value_1, i:Proc_2"
	Fired 124 times	- Rule "Write data, on DACKemw, v:Value_1, i:Proc_3"
	Fired 16 times	- Rule "Write data, on DACK, v:Value_1, i:Proc_1"
	Fired 73 times	- Rule "Write data, on DACK, v:Value_1, i:Proc_2"
	Fired 124 times	- Rule "Write data, on DACK, v:Value_1, i:Proc_3"
	Fired 3 times	- Rule "Invalidate, on DACKemw, v:Value_1, i:Proc_1"
	Fired 25 times	- Rule "Invalidate, on DACKemw, v:Value_1, i:Proc_2"
	Fired 126 times	- Rule "Invalidate, on DACKemw, v:Value_1, i:Proc_3"
	Fired 34 times	- Rule "Trying to write data, v:Value_1, i:Proc_1"
	Fired 41 times	- Rule "Trying to write data, v:Value_1, i:Proc_2"
	Fired 134 times	- Rule "Trying to write data, v:Value_1, i:Proc_3"
	Fired 126 times	- Rule "Write data, v:Value_1, i:Proc_1"
	Fired 75 times	- Rule "Write data, v:Value_1, i:Proc_2"
	Fired 25 times	- Rule "Write data, v:Value_1, i:Proc_3"

```

Verification of Futurebus+ Cache Coherence Protocol can be found
in [previous post](https://amutheezan.com/computer%20architecture/futurebus+).
