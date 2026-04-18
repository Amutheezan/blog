---
type: posts
title: Verification of Futurebus+ Cache Coherence Protocol
author: Amutheezan Sivagnanam
category: Computer Architecture
date: 2021-05-30
tags:
- architecture
- futurebus+
- murphi

---

In this post, I share a simple verification for the Futurebus+ cache coherence protocol as part of an assignment for the
**COSC 6385** course at the University of Houston.

## Initial Setup & Steps

Initially clone the [repository](https://github.com/Amutheezan/futurebus) and go to the code ```main``` directory and
execute the following two command to start the docker instances (make sure docker is already installed in your computer
and started; check here for installation steps of [docker](https://docs.docker.com/get-docker/).

```docker-compose build```

```docker-compose run ubuntu bash```

**Note**: the docker implementation and murphi3.1 integration obtained from the
following [repository](https://github.com/adnaneGdihi/fixed_murphi3.1).

After that you will get into the bash, follow the steps shown below in the ubuntu bash,

* First go to the directory ```Murphi3.1/src``` and compile the Murphi using ```make``` command (if it not already
  generated). And optionally make the executable access using the command ```chmod +x Murphi3.1/src/mu```, if you in the
  home directory.
* Then go to the directory ```verification``` and obtain the C file for the Futurebus+ verification
  using ```./../Murphi3.1/src/mu futurebus.m``` command (if it not already generated). This will generate the C
  file ```futurebus.C```.
* Thereafter, compile the generate file using the command ```make futurebus```, this eventually generate the executable
  program.
* Finally, run the executable using the following command ```./futurebus```

### OR

you can perform these entire steps by simply go to the ```verification``` directory and run the command ```sh run.sh```.

## Verification using Murphi3.1

Please note that Murphi3.1 come installed with docker image so no need to worry about downloading and setting up.

I structure the code as follows,

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

```c
 var
	proc_state: array[Proc] of ProcState;
	transaction_flag: boolean;
	last_write: Value;
	one_flag: boolean;
	more_flag: boolean;
	send_msg: Message;
```

* procedures - contains the function used by the verification.

```c
procedure SendMessage(msg: Message; i: Proc);
begin
	switch msg.mtype
		case ReadShared:
			if (proc_state[i].state = FB_EM | 
				proc_state[i].state = FB_EU) then
				proc_state[i].state :=  FB_SU;
			endif;

		case ReadModified:
			if (proc_state[i].state = FB_SU | 
				proc_state[i].state = FB_PR | 
				proc_state[i].state = FB_PSU | 
				proc_state[i].state = FB_PEMR | 
				proc_state[i].state = FB_EU) then
				proc_state[i].state :=  FB_I;
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

* ruleset - define the set of rules that can be used to model the systems.

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
			endrule;
		
			rule "Trying to write data"
				(p.state = FB_I)
			==>
				p.state :=  FB_PW;

				UpdateSignals();
			endrule;

			rule "Trying to write data (pending)"
				(p.state = FB_PW)
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
  system.

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
issues found and other statistics.

```
Protocol: futurebus

Algorithm:
	Verification by breadth first search.
	with symmetry algorithm 3 -- Heuristic Small Memory Normalization
	with permutation trial limit 10.

Memory usage:

	* The size of each state is 96 bits (rounded up to 12 bytes).
	* The memory allocated for the hash table and state queue is
	  8 Mbytes.
	  With two words of overhead per state, the maximum size of
	  the state space is 487811 statesn the above code, I structure code in the following structure,
\begin{itemize}
    \item constants - to define constants such as number of processor, number of values
    \item type - define types such as processor states using enumeration which represents all the states in Futurebus+ proctcol, message types as enumeration which includes different message used to send between the states either bus or cpu call, finally the message type.
	   * Use option "-k" or "-m" to increase this, if necessary.
	* Capacity in queue for breadth-first search: 48781 states.
	   * Change the constant gPercentActiveStates in mu_prolog.inc
	     to increase this, if necessary.

Warning: No trace will not be printed in the case of protocol errors!
         Check the options if you want to have error traces.

Result:

	Invariant "only one processor in EM state" failed.

State Space Explored:

	56 states, 184 rules fired in 0.10s.

Analysis of State Space:

	There are rules that are never fired.
	If you are running with symmetry, this may be why.  Otherwise,
	please run this program with "-pr" for the rules information.
```

Next I provides the output of error traces of Futurebus+ traces of cache coherence protocol using the command
suffix ```-tv```. This provides the the reasons why the error happen (i.e two processors go to the exclusive states).

```
Protocol: futurebus

Algorithm:
	Verification by breadth first search.
	with symmetry algorithm 3 -- Heuristic Small Memory Normalization
	with permutation trial limit 10.

Memory usage:

	* The size of each state is 96 bits (rounded up to 12 bytes).
	* The memory allocated for the hash table and state queue is
	  8 Mbytes.
	  With two words of overhead per state, the maximum size of
	  the state space is 487811 states \item variables - the global variables represents the system.
	   * Use option "-k" or "-m" to increase this, if necessary.
	* Capacity in queue for breadth-first search: 48781 states.
	   * Change the constant gPercentActiveStates in mu_prolog.inc
	     to increase this, if necessary.

The following is the error trace for the error:

	Invariant "only one processor in EM state" failed.

Startstate Startstate 0 fired.
proc_state[Proc_1].state:FB_I
proc_state[Proc_1].value:Undefined
proc_state[Proc_2].state:FB_I
proc_state[Proc_2].value:Undefined
proc_state[Proc_3].state:FB_I
proc_state[Proc_3].value:Undefined
transaction_flag:false
last_write:Undefined
one_flag:false
more_flag:false
send_msg.mtype:Undefined
send_msg.value:Undefined

Rule Trying to write data, v:Value_1, i:Proc_1 fired.
proc_state[Proc_1].state:FB_PW

Rule Trying to write data, v:Value_1, i:Proc_2 fired.
proc_state[Proc_2].state:FB_PW

Rule Write data, on DACKemw, v:Value_1, i:Proc_1 fired.
proc_state[Proc_1].state:FB_EM
proc_state[Proc_1].value:Value_1
proc_state[Proc_2].value:Value_1
transaction_flag:true
last_write:Value_1
send_msg.mtype:ReadModified
send_msg.value:Value_1

Rule Write data, on DACKemw, v:Value_1, i:Proc_2 fired.
The last state of the trace (in full) is:
proc_state[Proc_1].state:FB_EM
proc_state[Proc_1].value:Value_1
proc_state[Proc_2].state:FB_EM
proc_state[Proc_2].value:Value_1
proc_state[Proc_3].state:FB_I
proc_state[Proc_3].value:Undefined
transaction_flag:true
last_write:Value_1
one_flag:false
more_flag:false
send_msg.mtype:ReadModified
send_msg.value:Value_1

End of the error trace.

Result:

	Invariant "only one processor in EM state" failed.

State Space Explored:

	56 states, 184 rules fired in 0.10s.

Analysis of State Space:

	There are rules that are never fired.
	If you are running with symmetry, this may be why.  Otherwise,
	p \item procedures - contains the function used by the verification.
    \item rulease run this program with "-pr" for the rules information.

```

Finally, to observe what are rule fired, I obtain the output of rule firing of Futurebus+ traces of cache coherence
protocol using the command suffix ```-pr```.

```
Protocol: futurebus

Algorithm:
	Verification by breadth first search.
	with symmetry algorithm 3 -- Heuristic Small Memory Normalization
	with permutation trial limit 10.

Memory usage:

	* The size of each state is 96 bits (rounded up to 12 bytes).
	* The memory allocated for the hash table and state queue is
	  8 Mbytes.
	  With two words of overhead per state, the maximum size of
	  the state space is 487811 statet - define the set of rules that can be used to model the systems.
	   * Use option "-k" or "-m" to increase this, if necessary.
	* Capacity in queue for breadth-first search: 48781 states.
	   * Change the constant gPercentActiveStates in mu_prolog.inc
	     to increase this, if necessary.

Warning: No trace will not be printed in the case of protocol errors!
         Check the options if you want to have error traces.

Result:

	Invariant "only one processor in EM state" failed.

State Space Explored:

	56 states, 184 rules fired in 0.10s.

Rules Information:

	Fired 0 times	- Rule "Read data from Exclusive states, i:Proc_1"
	Fired 4 times	- Rule "Read data from Exclusive states, i:Proc_2"
	Fired 4 times	- Rule "Read data from Exclusive states, i:Proc_3"
	Fired 4 times	- Rule "Go to Exclusive state on DACK, i:Proc_1"
	Fired 5 times	- Rule "Go to Exclusive state on DACK, i:Proc_2"
	Fired 3 times	- Rule "Go to Exclusive state on DACK, i:Proc_3"
	Fired 3 times	- Rule "Go to Shared state on DACKem, i:Proc_1"
	Fired 4 times	- Rule "Go to Shared state on DACKem, i:Proc_2"
	Fired 2 times	- Rule "Go to Shared state on DACKem, i:Proc_3"
	Fired 3 times	- Rule "Go to Shared state on DACK, i:Proc_1"
	Fired 4 times	- Rule "Go to Shared state on DACK, i:Proc_2"
	Fired 2 times	- Rule "Go to Shared state on DACK, i:Proc_3"
	Fired 0 times	- Rule "Go to Shared state on DACK, i:Proc_1"
	Fired 0 times	- Rule "Go to Shared state on DACK, i:Proc_2"
	Fired 2 times	- Rule "Go to Shared state on DACK, i:Proc_3"
	Fired 0 times	- Rule "Go to Shared state on DACK, i:Proc_1"
	Fired 0 times	- Rule "Go to Shared state on DACK, i:Proc_2"
	Fired 1 times	- Rule "Go to Shared state on DACK, i:Proc_3"
	Fired 0 times	- Rule "Go to Pending Shared state, i:Proc_1"
	Fired 5 times	- Rule "Go to Pending Shared state, i:Proc_2"
	Fired 2 times	- Rule "Go to Pending Shared state, i:Proc_3"
	Fired 0 times	- Rule "Read data from EM, i:Proc_1"
	Fired 1 times	- Rule "Read data from EM, i:Proc_2"
	Fired 3 times	- Rule "Read data from EM, i:Proc_3"
	Fired 0 times	- Rule "Read shared data, i:Proc_1"
	Fired 2 times	- Rule "Read shared data, i:Proc_2"
	Fired 0 times	- Rule "Read shared data, i:Proc_3"
	Fired 0 times	- Rule "Moved to shared state from EMR on DACKem, i:Proc_1"
	Fired 0 times	- Rule "Moved to shared state from EMR on DACKem, i:Proc_2"
	Fired 0 times	- Rule "Moved to shared state from EMR on DACKem, i:Proc_3"
	Fired 4 times	- Rule "Trying to read data (pending), i:Proc_1"
	Fired 5 times	- Rule "Trying to read data (pending), i:Proc_2"
	Fired 5 times	- Rule "Trying to read data (pending), i:Proc_3"
	Fired 18 times	- Rule "Trying to read data, i:Proc_1"
	Fired 8 times	- Rule "Trying to read data, i:Proc_2"
	Fired 1 times	- Rule "Trying to read data, i:Proc_3"
	Fired 1 times	- Rule "Trying to write data (pending), v:Value_1, i:Proc_1"
	Fired 4 times	- Rule "Trying to write data (pending), v:Value_1, i:Proc_2"
	Fired 12 times	- Rule "Trying to write data (pending), v:Value_1, i:Proc_3"
	Fired 18 times	- Rule "Trying to write data, v:Value_1, i:Proc_1"
	Fired 8 times	- Rule "Trying to write data, v:Value_1, i:Proc_2"
	Fired 1 times	- Rule "Trying to write data, v:Value_1, i:Proc_3"
	Fired 1 times	- Rule "Write data, on DACKemw, v:Value_1, i:Proc_1"
	Fired 4 times	- Rule "Write data, on DACKemw, v:Value_1, i:Proc_2"
	Fired 12 times	- Rule "Write data, on DACKemw, v:Value_1, i:Proc_3"
	Fired 1 times	- Rule "Write data, on DACK, v:Value_1, i:Proc_1"
	Fired 4 times	- Rule "Write data, on DACK, v:Value_1, i:Proc_2"
	Fired 11 times	- Rule "Write data, on DACK, v:Value_1, i:Proc_3"
	Fired 0 times	- Rule "Invalidate, on DACKemw, v:Value_1, i:Proc_1"
	Fired 0 times	- Rule "Invalidate, on DACKemw, v:Value_1, i:Proc_2"
	Fired 0 times	- Rule "Invalidate, on DACKemw, v:Value_1, i:Proc_3"
	Fired 0 times	- Rule "Trying to write data, v:Value_1, i:Proc_1"
	Fired 0 times	- Rule "Trying to write data, v:Value_1, i:Proc_2"
	Fired 3 times	- Rule "Trying to write data, v:Value_1, i:Proc_3"
	Fired 0 times	- Rule "Write data, v:Value_1, i:Proc_1"
	Fired 5 times	- Rule "Write data, v:Value_1, i:Proc_2"
	Fired 4 times	- Rule "Write data, v:Value_1, i:Proc_3"

```

In the [next post](https://amutheezan.com/computer%20architecture/fixfuturebus+/), I will write about how to apply a
simple fix for Futurebus+ protocol to avoid multiple processers get into exclusive state. 
