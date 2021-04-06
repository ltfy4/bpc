# bpc_dimensionread
*Used to read dimensions in BPC project codes

* ---------------------------------------------------------------------------------------+
* | ->BPC_READ_DIMENSION---> Parameters
* +--------------------------------------------------------------------------------------+
* | [--->] I_APPSET_ID                    TYPE        UJ_APPSET_ID
* | [--->] I_DIM_NAME                     TYPE        UJ_DIM_NAME
* | [--->] I_BASE_MEMBER_OF               TYPE        UJ_DIM_MEMBER(optional)
* | [<---] ER_DATA                        TYPE REF TO DATA
* | [<---] ET_MESSAGE                     TYPE        
* +--------------------------------------------------------------------------------------

  method bpc_read_dimension.

    data: lo_dim       type ref to cl_ujk_model,
          et_bas_list  type uja_t_dim_member,
          lx_exception type ref to cx_root,
          lo_dataref   type ref to data,
          lo_dim_data  type ref to if_uja_dim_data,
          lo_reader    type ref to if_uja_md_reader,
          lo_query     type ref to cl_uja_md_query_opt,
          ls_message   like line of et_message.

    field-symbols: <lt_md_data> type standard table,
                   <ls_md_data> type any,
                   <id>         type any.

    if i_base_member_of is not initial .
      try.
          call method cl_ujk_model=>get_children
            exporting
              i_appset_id  = i_appset
              i_dim        = i_dim
              i_parent_mbr = i_base_member_of
            importing
              et_bas_list  = et_bas_list.
        catch cx_uj_static_check into lx_exception .
          ls_message-message = lx_exception->get_longtext( ).
          append ls_message to et_message.
      endtry.
    endif .

    try.
        lo_dim_data = cl_uja_admin_mgr=>create_dim_ref( i_appset_id = i_appset i_dim_name = i_dim ).
        create object lo_query exporting io_dim = lo_dim_data.

        lo_query->select_all_attr( exporting if_inc_non_display = abap_false
                                             if_inc_slt         = abap_true
                                             if_inc_txt         = abap_true
                                             if_inc_generate    = abap_false ).
        lo_query->select_all_hier( ).

        lo_reader = lo_dim_data->get_md_reader( ).
        lo_reader->read( exporting io_read_opt = lo_query
                         importing er_data = lo_dataref  ).

        assign lo_dataref->* to <lt_md_data>.

      catch cx_uja_admin_error cx_uj_obj_not_found cx_uj_static_check into lx_exception .
        ls_message-message = lx_exception->get_longtext( ).
        append ls_message to et_message.
    endtry.

    loop at <lt_md_data> assigning <ls_md_data> .
      assign component 'ID' of structure <ls_md_data> to <id> .
      if <id> is assigned .
        read table et_bas_list transporting no fields with key table_line = <id> .
        if sy-subrc ne 0 and i_base_member_of is not initial .
          delete table <lt_md_data> from <ls_md_data> .
        endif .
      endif .
    endloop .

    get reference of <lt_md_data> into er_data .

  endmethod.
