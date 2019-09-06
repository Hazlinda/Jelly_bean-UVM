# Jelly_bean-UVM

This post I will write a simple tutorial on verification methodology for Jelly_bean. This example I took form this link:
http://cluelogic.com/2011/07/uvm-tutorial-for-candy-lovers-overview/

# DUT

	    //////////////////////
	    //////////////////////
	    ///////DUT////////////
	    //////////////////////
	    //////////////////////

	    `include "uvm_macros.svh"
	    `include "jelly_bean_if.sv"
	    `include "jelly_bean_pkg.sv"

	     module jelly_bean_taster( jelly_bean_if.slave_mp jb_slave_if );
	      import jelly_bean_pkg::*;

		always @ ( posedge jb_slave_if.clk ) begin
		    if ( jb_slave_if.flavor == jelly_bean_transaction::CHOCOLATE &&
			jb_slave_if.sour ) begin
			jb_slave_if.taste <= jelly_bean_transaction::YUCKY;
		end else begin
			jb_slave_if.taste <= jelly_bean_transaction::YUMMY;
		end
	      end
	    endmodule: jelly_bean_taster



# TRANSACTION

	    ///////////////////////
	    ///////////////////////
	    /////Transaction.sv////
	    ///////////////////////
	    ///////////////////////
	    class jelly_bean_transaction extends uvm_sequence_item;

	      typedef enum bit [2:0] {NO_FLAVOR, APPLE, BLUEBERRY, BUBBLE_GUM, CHOCOLATE} flavor_e;
	      typedef enum bit [1:0] {RED, GREEN, BLUE} color_e;
	      typedef enum bit [1:0] {UNKNOWN, YUMMY, YUCKY} taste_e;

	     ///////////////////////////////////
	     //declaring the transaction items//
	     ///////////////////////////////////
	      rand flavor_e flavor;
	      rand color_e color;
	      rand bit sugar_free;
	      rand bit sour;
	      taste_e taste;

	      ////////////////////////////////////////////////////////
	      //constraint, to generate flavor and color//////////////
	      ////////////////////////////////////////////////////////
	      constraint flavor_color_c{
		flavor != NO_FLAVOR;
		flavor == APPLE -> color != BLUE;
		flavor == BLUEBERRY -> color == BLUE;
	      }

	      function new(string name = "");
		super.new(name);
	      endfunction : new

	      `uvm_object_utils_begin(jelly_bean_transaction)
		`uvm_field_enum(flavor_e, flavor, UVM_ALL_ON)
		`uvm_field_enum(color_e, color, UVM_ALL_ON)
		`uvm_field_int(sugar_free, UVM_ALL_ON)
		`uvm_field_int(sour, UVM_ALL_ON)
		`uvm_field_enum(taste_e, taste, UVM_ALL_ON)
	      `uvm_object_utils_end

	    endclass : jelly_bean_transaction  

	    class sugar_free_jelly_bean_transaction extends jelly_bean_transaction;
	      `uvm_object_utils(sugar_free_jelly_bean_transaction)

	      /////////////////////////////////////////////////
	      //constraint, to generate sugarfree//////////////
	      /////////////////////////////////////////////////
	      constraint sugar_free_c{
		sugar_free == 1;
	      }

	      function new(string name = "");
		super.new(name);
	      endfunction : new
	    endclass : sugar_free_jelly_bean_transaction



# INTERFACE

	    ///////////////////////
	    ///jely_bean_if.sv/////
	    ///////////////////////
	    ///////////////////////

	    interface jelly_bean_if( input bit clk );
	    ////////////////////////
	    //declaring the signal//
	    ////////////////////////
	      logic [2:0] flavor;
	      logic [1:0] color;
	      logic       sugar_free;
	      logic       sour;
	      logic [1:0] taste;

	    ////////////////////////////////////
	    //////master clocking block/////////
	    ////////////////////////////////////
	      clocking master_cb @ ( posedge clk );
		  default input #1step output #1ns;
		  output flavor, color, sugar_free, sour;
		  input  taste;
	      endclocking: master_cb

	    ////////////////////////////////////
	    //////slave clocking block//////////
	    ////////////////////////////////////
	      clocking slave_cb @ ( posedge clk );
		  default input #1step output #1ns;
		  input  flavor, color, sugar_free, sour;
		  output taste;
	      endclocking: slave_cb

	      //////////////////////////////////////////
	      //Declaring modport for master and slave//
	      //////////////////////////////////////////
	      modport master_mp( input clk, taste, output flavor, color, sugar_free, sour );
	      modport slave_mp ( input clk, flavor, color, sugar_free, sour, output taste );
	      modport master_sync_mp( clocking master_cb );
	      modport slave_sync_mp ( clocking slave_cb  );

	    endinterface: jelly_bean_if

# Test
	////////////////////////
	////////////////////////
	//jelly_bean_test.sv////
	////////////////////////
	////////////////////////
	class jelly_bean_test extends uvm_test;
		`uvm_component_utils (jelly_bean_test)

		jelly_bean_env jb_env;

		function new(string name, uvm_component parent);	
			super.new(name, parent);
			endfunction : new

		function void build_phase(uvm_phase phase);
			super.build_phase(phase);
			begin	
				jelly_bean_configuration jb_cfg;

				jb_cfg = new;
				assert (jb_cfg.randomize());
				uvm_config_db#(jelly_bean_configuration)::set(this, "*", "config", jb_cfg);
				jelly_bean_transaction::type_id::set_type_override(
					sugar_free_jelly_bean_transaction::get_type());
				jb_env = jelly_bean_env::type_id::create("jb_env", this);
			end
		endfunction:buid_phase


		task run_phase (uvm_phase phase);
			gift_boxes_jelly_bean_sequence jb_seq;

			phase.raise_objection(this);
			jb_seq = gift_boxes_jelly_bean_sequence::type_id::create("jb_seq");
			assert (jb_seq.randomize());
			`uvm_info("jelly_bean_test", {"\n", jb_seq.sprint()}, UVM_LOW)
			jb_seq.start(jb_env.jb_agt.jb_seqr);
			#10ns;
			phase.drop_objection(this);
		endtask: run_phase
	endclass: jelly_bean_test

# Emvironment

	/////////////////////
	/////////////////////
	//jelly_bean_env.sv//
	/////////////////////
	/////////////////////
	class jelly_bean_env extends uvm_component;
		`uvm_component+utils(jelly_bean_env)

		jelly_bean_fc_subscriber jb_fc_sub;
		jelly_bean_scoreboard jb_scb;
		jelly_bean_agent jb_agt;

		function new (string name, uvm_component parent);
			super.new(name, parent);
		endfunction : new

		function void build_phase(uvm_phase phase);
			super.build_phase(phase);
			jb_fc_sub = jelly_bean_fc_subscriber::type_id::create("jb_fc_sub", this);
			jb_scb = jelly_bean_scoreboard::type_id::create("jb_scb", this);
			jb_agt = jelly_bean_agent::type_id::create("jb_agt", this);
		endfunction : build_phase

		function void connect_phase (uvm_phase phase);
			super.connect_phase(phase);
			jb_agt.jb_ap.connect(jb_fc_sub.analysis_export);
			jb_agt.jb_ap.connect(jb_scb.jb_analysis_export);
		endfunction : connect_phase
	endclass : jelly_bean_env


# Agent

	///////////////////////
	///////////////////////
	//jelly_bean_agent.sv//
	///////////////////////
	///////////////////////
	`ifndef JELLY_BEAN_AGENT
	`define JELLY_BEAN_AGENT
	`include "jelly_bean_sequencer.sv"
	`include "jelly_bean_driver.sv"
	`include "jelly_bean_monitor.sv"

	class jelly_bean_agent extends uvm_agent;
		`uvm_component_utils(jelly_bean_agent)

		uvm_analysis_port#( jelly_bean_transaction ) jb_ap;

		jelly_bean_sequencer jb_seqr;
		jelly_bean_driver jb_drvr;
		jelly_bean_monitor jb_mon;

		function new ( string name, uvm_component parent );
			super.new( name, parent );
		endfunction:new

		function void build_phase( uvm_phase phase );
			super.build_phase ( phase );

			jb_ap = new ( .name( "jb_ap" ), .parent(this));
			jb_seqr = jelly_bean_sequencer::type_id::create( .name("jb_seqr"), .parent( this ));
			jb_drvr = jelly_bean_driver	  ::type_id::create( .name("jb_drvr"), .parent( this ));
			jb_mon = jelly_bean_monitor	  ::type_id::create( .name("jb_mon"), .parent( this ));
		endfunction:build_phase

		function void connect_phase( uvm_phase phase );
			super.connect_phase( phase );
			jb_drvr.seq_item_port.connect( jb_seqr.seq_item_export );
			jb_mon.jb_ab.connect( jb_ap );
		endfunction:connect_phase
	endclass: jelly_bean_agent

	`endif





# Driver

	/////////////////////////
	/////////////////////////
	//jelly_bean_driver.sv///
	/////////////////////////
	/////////////////////////

	class jelly_bean_driver extends uvm_driver #(jelly_bean_transaction);
		`uvm_component_utils(jelly_bean_driver)
		virtual jelly_bean_if vif; //virtual declaration

		function new(string name, uvm_component parent);
			super.new(name, parent);
		endfunction:new

		function void build_phase (uvm_phase phase);
			super.build_phase(phase);
			if(!uvm_config_db#(virtual jelly_bean_if)::get(this, "","vif", vif))
				`uvm_fatal("jelly_bean_driver", "virtual interface must be set for vif")
		endfunction: build_phase

		task run_phase(uvm_phase phase);
			jelly_bean_transaction jb_tx;
			forever begin
				@(vif.master_cb);
				vif.master_cb.flavor <= NO_FLAVOR;
				seq_item_port.get_next_item(jb_tx);
				vif.master_cb.flavor	<= jb_tx.flavor;
				vif.master_cb.color		<= jb_tx.color;
				vif.master_cb.sugar_free<= jb_tx.sugar_free;
				vif.master_cb.sour 		<= jb_tx.sour;
				seq_item_port.item_done();
			end
		endtask : run_phase

	endclass : jelly_bean_driver
	


    
