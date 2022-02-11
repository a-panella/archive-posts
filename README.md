# archive-posts
a repo saving some of the best old xilinx/vivado posts

#the guide to virtual clocks and timing, by avrum
https://forums.xilinx.com/t5/Timing-Analysis/How-to-constraint-Same-Edge-capture-edge-aligned-DDR-input/m-p/646009#M8411

Avrum: OK...

This is going to be long and complicated. My suggestion, if you really want to follow this, grab a piece of paper and sketch both the circuit and the waveforms as I describe them...

I know your period is large, but I am going to base my constraints on a 10ns DDR interface with a 0.4ns clock/data skew. For the moment, I will assume the clock is a perfect 50% duty cycle.

First, lets understand how the static timing analysis sees these elements. Lets start with the IDDR. From the static timing analysis point of view, this is really two flip-flops; one clocked on the rising edge of the clock and one clocked on the falling edge of the clock.

Now lets get started on constraints. First I am going to define the clock. As will become evident later, I will need two different clocks for this - one clock for driving the FPGA and one for driving the "external device" (used for the set_input_delay). These will be identical in terms of their properties (period, phase, duty cycle) but I need two different ones for the constraints. In Vivado all clocks are related by default, so even though I define two clocks, they are effectively the same if they have the same attributes

create_clock -name fpga_clk -period 10 [get_ports fpga_clk_pin] create_clock -name virt_clk -period 10

Now I am going to define the input delays. I will revisit this later, but for now lets use the "simple" ones (these will eventually be the ones for the "correct way" solution)

set_input_delay -clock virt_clk 0.4 [get_ports data_in] set_input_delay -clock virt_clk -0.4 -min [get_ports data_in] set_input_delay -clock virt_clk 0.4 [get_ports data_in] -clock_fall -add_delay set_input_delay -clock virt_clk -0.4 -min [get_ports data_in] -clock_fall -add_delay

Again, lets look at the structure of this. From the STA point of view, we are defining two external flip-flops; one clocked on the rising edge of the clock and one clocked on the falling edge of the clock.

Thus (and you may want to draw this), we have two external FFs (one on each edge) coming together (don't worry about any form of MUX), then going through the data_in port and ending at the IDDR which are two internal flip-flops (one on each edge). So from the static timing point of view, we have FOUR paths (yes FOUR paths)

a) from rising edge external FF to falling edge internal FF b) from falling edge external FF to rising edge internal FF c) from rising edge external FF to rising edge internal FF d) from falling edge external FF to falling edge internal FF Two of these are clearly false (or at least redundant), but all 4 of these will become important later.

Now lets look at how the static timing engine determines launch and capture edges.

The constraints above define two windows of valid timing. The first window is launched by the rising edge of the external FF (since it is described by the set_input_delay commands without the -clock_fall); it starts at time 0.4, and ends at time 4.6 (this isn't technically true, but its good enough for this description). The second window is (created by the set_input_delay with the -clock_fall) starts at 5.4ns and ends at 9.6.

Lets assume that you will use an IDELAY on the clock coming in to the FPGA and delay it by 2.5ns (to place it in the middle of the eye - again, this probably isn't the correct thing to do, since the FPGA has an asymmetric setup/hold requirement, but its good enough for illustration).

So draw the launch clock, which is the virtual clock (virt_clk), it has rise edges at 0ns and 10ns and fall edges at 5ns and 15ns.

Draw the data windows below that, as described above (0.4 to 4.6, and 5.4 to 9.6)

Draw the fpga clock (fpga_clk) below that (which has the same edges as the virtual clock, edges at 0, 5, 10, 15)

Draw the delayed internal clock, with rising edges at 2.5 and 12.5, and falling edges at 7.5 and 17.5.

Now, lets look at the "rule" for multiple clocks in Vivado. To determine which edge is the capture edge, Vivado looks at the un-propagated clocks (thus fpga_clk and virt_clk). If data is launched on one edge, it is by definition captured on the un-propagated clock edge that follows it in time.

Now we have to look at all 4 of the above paths

a) launched from rising edge virt_clk @0ns, the falling edge of fpga_clk that follows it is at 5ns b) launched from falling edge virt_clk @5ns, the rising edge of fpga_clk that follows it is at 10ns c) launched from rising edge virt_clk @0ns, the rising edge of fpga_clk that follows it is at 10ns d) launched from falling edge virt_clk @5ns, the falling edge of fpga_clk that follows it is at 15ns The actual capture would happen on the propagated clock 2.5ns later, but that's irrelevent for determining the edges.

Clearly if a) passes its setup check, then so will c) (since it has 5ns more time), so the c) path is redundant. The same is true of d); if the b) path passes, the d) path will.

So looking at this, none of these are what we want. We really want

launched from the rising edge of virt_clk @0ns, captured at the propagated fpga_clk at 2.5ns, which is the un-propagated clock of fpga_clk at 0ns the same for the falling edge (launched at 5ns, captured at 5ns). So, we have two ways of doing this - the "cheating way", which is a bit simpler, and the "correct way".

Lets start with the "cheating" way, which is what is done by the language templates in Vivado - do "Windows -> Language Templates" and select "XDC -> Timing Constraints -> Input Delay Constraints -> Source Synchronous -> Edge Alinged (Clock directly to FF) -> Dual Data Rate (DDR).

This mechanism uses the definitions above to "lie" to the tool. Lets look at path a). We know that the tool assumes that the window defined by the risinge edge of virt_clk @0ns will be captured by the falling edge of fpga_clk at @5ns. But, we want the window at 5.4 to 9.6 to be the window captured by the falling edge of fpga_clk, so lets lie. Instead of having the window generated by the rising edge of virt_clk be from 0.4 to 4.6, lets lie to the tool at say that the rising edge of virt_clk really generates the window at 5.4 to 9.6. This is done not by the set_input_delay commands shown above, but by a set with a constant 5ns (1/2 of the clock period) added to all timing. Thus

set_input_delay -clock virt_clk 5.4 [get_ports data_in] set_input_delay -clock virt_clk 4.6 -min [get_ports data_in] set_input_delay -clock virt_clk 5.4 [get_ports data_in] -clock_fall -add_delay set_input_delay -clock virt_clk 4.6 -min [get_ports data_in] -clock_fall -add_delay

This is what the language template and the clocking wizard get you to do; if you use the language template with all 4 skew values being 0.4 and the input_clock_period as 10, you will get the above constraints.

So now, the rising edge of virt_clk at time 0 will generate the window at 5.4 to 9.6. By the rule (for path a), since it is launched by virt_clk at time 0, it will be captured by fpga_clk at time 5ns. This will be delayed internally to 7.5ns, so the window from 5.4 to 9.6 will be captured at 7.5. This is what we want!

So paths a) and b) will give you what you want, paths c) and d) are redundant - they will still be timed but if a) and b) pass, then so will c) and d).

So, what's wrong with this. First, it is lying - its not really what is happening; the rising edge of virt_clk isn't really creating the window at 5.4 to 9.6, its generating the window at 0.4 to 4.6. It also gives you (at least) one significant problem; what if the clock duty cycle isn't 50%? How do you continue the lie to adjust for this fact? It can be done, but it gets increasingly messy.

OK, so now the right way. I am warning you, its pretty complicated...

So, lets start back at the original constraints; the create_clock for fpga_clk and virt_clk and the original set of set_input_delays (without the 1/2 clock period added).

As I said before, these aren't what you want. The default capture edges are not what you want. For example, for the launch edge at 0ns, you want to get it to use the capture edge at 0ns, instead of the default one, which will be at 5ns (for path a) and at 10ns (for path c).

So, how do we change this. The answer is (drum roll please!) - with the set_multicycle_path command!

This is, in fact, exactly what the set_multicycle_path command does - it changes which edge is used for capture. Normally, the capture edge is one clock after the launch edge (which is an edge count of 1). When we do set_multicycle_path 2 for a path, we are changing the capture edge to be 2 clocks later (instead of the default of one edge later). This is why it is called "multi-cycle" - we usually use it to give a path multple cycles to propagate.

But once we realize that what it is doing is changing the capture edge (at least by default), we can use this here. But instead of wanting to move the capture edge forward (from 1 to 2) we want to move it backward (from 1 to 0). So, we use this command

set_multicycle_path 0 -from [get_clocks virt_clk] -to [get_clocks fpga_clk]

This effectively pulls the capture edge back by one full clock period.

So now lets look at our four paths

a) Launch at 0ns to capture a -5ns (from 5ns pulled back one complete clock period) b) Launch at 5ns to capture at 0ns (from 10ns) c) Launch at 0ns to capture at 0ns (from 10ns) - this is the correct one! d) Launch at 5ns to capture at 5ns (from 15ns) - this is the other correct one Clearly, though, now a) and b) are not just incorrect, but they will fail. So we need to disable them

set_false_path -setup -rise_from [get_clocks virt_clk] -fall_to [get_clocks fpga_clk]; # diable a) set_false_path -setup -fall_from [get_clocks virt_clk] -rise_to [get_clocks fpga_clk]; # diable b)

This is (by the way) why I need the two clocks to be separate; I need the -rise_* and -fall_* to refer to clock edges, so I need separate clocks for this.

So now we have the correct setup checks. The hold checks are even more complicated; I won't go through the details, but they need these:

set_multicycle_path -1 -hold -from [get_clocks virt_clk] -to [get_clocks fpga_clk] set_false_path -hold -rise_from [get_clocks virt_clk] -rise_to [get_clocks fpga_clk]; set_false_path -hold -fall_from [get_clocks virt_clk] -fall_to [get_clocks fpga_clk];

Now we have all the setup and hold checks being done correctly! (Yay!)

But I said I would come back to the duty cycle...

Lets say the duty cycle of the clock is unknown 40/60 to 60/40. This means the falling edge of the clock can be anywhere between 4ns and 6ns (instead of 5ns).

You could deal with this by just effectively increasing the 0.4ns "skews" and add an additional 1ns to them (which would probably work), but you can also do this "correctly"

NOTE: This didn't fully work in an earlier version of Vivado, and I haven't tested it recently...

You can change the duty cycle of the clock using the set_clock_latency command. This command allows you to add latency to the clock on both the rising edge and falling edge and for both min and max delays. So we can do

set_clock_latency -source -fall -min -1 [get_clocks {virt_clk fpga_clk}] set_clock_latency -source -fall -max 1 [get_clocks {virt_clk fpga_clk}]

(thus adding -1ns min and +1ns max to the falling edge while leaving the rising edges alone)

So, that's it. Simple, eh?

Avrum




#Constraining CDCs
## (so that you don't have to use XPM cdcs, which are not synthisable for GHDL)

Asker: or can I just tell Vivado that these 2 clocks are unrelated, so I don't need to specify the paths in detail?

Avrum: Absolutely not.

First lets look at 1) the path from the first sync FF to the 2nd FF (the path that can go metastable). According to Xilinx the only thing that is necessary is the ASYNC_REG property.

In the past I would have constrained this path to be "small". The whole point of the 2 back to back flip-flops is to ensure that there is "time" for the metastability of the first flip-flop to resolve before the second flip-flop samples it. With no constraint, the tools can take just shy of one full destination clock period for the routing - if it does this, then it leaves no time for metastability resolution.

The ASYNC_REG property is supposed to solve this by ensuring that the FFs stay "near" eachother - in the same slice if possible. If they are really in the same slice, then I can't see any way for the route between them to be anything but really short. If they are in neighboring slices (which should only happen if they can't be packed into the same slice, which shouldn't happen here), in theory there still could be significant routing delays between them (very unlikely, but possible).

So, in the past, and maybe even now, I would constrain them with

set_max_delay 1.5 -from <meta_flop> -to <dst_flop>

I know that 1.5ns (including clock skew) is enough for two reasonably closely placed flip-flops, which leaves the remainder of the clock period for metastability, Note that there is no -datapath_only on this constraint.

For 2) the path between the 2 data register latches in each domain - you must constrain this.

The synchronizer of the toggle event takes 1-2 dst clocks, which means that the sampling of the data_dst_clk register takes place 2-3 dst clock periods after the toggle of src_send_toggle_src_clk plus the routing delay between src_send_toggle_src_clk and event_toggle_meta.

Without constraints (i.e. declaring the clocks unrelated) the routing between src_din_reg and data_dst_clk is unconstrained (as is the path from src_send_toggle_src_clk to event_toggle_meta). This means that it can (for example) take 2ns for the path between src_send_toggle_src_clk to event_toggle_meta, and 3000ns for the path between src_din_reg and data_dst_clk. So data_dst_clk is updated 2-3 dst_clk periods plus 2ns after the edge of src_clk that causes src_send_toggle_src_clk to toggle, but the data only arrives at data_dst_clk 3000ns after that edge of src_clk - clearly your synchronizer will fail under these conditions.

So you absolutely need constraints to limit the skew between these paths - the best way to do that is with a set_max_delay -datapath_only with a "reasonable" time (say one dst_clk period, or even one period of the smaller clock) on each path that crosses the clock domain boundary.
