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

      rand flavor_e flavor;
      rand color_e color;
      rand bit sugar_free;
      rand bit sour;
      taste_e taste;
  
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


   ///////////////////
    
