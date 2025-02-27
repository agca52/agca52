## Hi there 👋

<!--
**agca52/agca52** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m Eagle_Freedom ...
- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m about teamwork and projects   ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...


#pragma version =0.2.0;
;; Wallet smart contract with plugins

(slice, int) dict_get?(cell dict, int key_len, slice index) asm(index dict key_len) "DICTGET" "NULLSWAPIFNOT";
(cell, int) dict_add_builder?(cell dict, int key_len, slice index, builder value) asm(value index dict key_len) "DICTADDB";
(cell, int) dict_delete?(cell dict, int key_len, slice index) asm(index dict key_len) "DICTDEL";

() recv_internal(int msg_value, cell in_msg_cell, slice in_msg) impure {
  var cs = in_msg_cell.begin_parse();
  var flags = cs~load_uint(4);  ;; int_msg_info$0 ihr_disabled:Bool bounce:Bool bounced:Bool
  if (flags & 1) {
    ;; ignore all bounced messages
    return ();
  }
  if (in_msg.slice_bits() < 32) {
    ;; ignore simple transfers
    return ();
  }
  int op = in_msg~load_uint(32);
  if (op != 0x706c7567) & (op != 0x64737472) { ;; "plug" & "dstr"
    ;; ignore all messages not related to plugins
    return ();
  }
  slice s_addr = cs~load_msg_addr();
  (int wc, int addr_hash) = parse_std_addr(s_addr);
  slice wc_n_address = begin_cell().store_int(wc, 8).store_uint(addr_hash, 256).end_cell().begin_parse();
  var ds = get_data().begin_parse().skip_bits(32 + 32 + 256);
  var plugins = ds~load_dict();
  var (_, success?) = plugins.dict_get?(8 + 256, wc_n_address);
  if ~(success?) {
    ;; it may be a transfer
    return ();
  }
  int query_id = in_msg~load_uint(64);
  var msg = begin_cell();
  if (op == 0x706c7567) { ;; request funds

    (int r_toncoins, cell r_extra) = (in_msg~load_grams(), in_msg~load_dict());

    [int my_balance, _] = get_balance();
    throw_unless(80, my_balance - msg_value >= r_toncoins);

    msg = msg.store_uint(0x18, 6)
             .store_slice(s_addr)
             .store_grams(r_toncoins)
             .store_dict(r_extra)
             .store_uint(0, 4 + 4 + 64 + 32 + 1 + 1)
             .store_uint(0x706c7567 | 0x80000000, 32)
             .store_uint(query_id, 64);
    send_raw_message(msg.end_cell(), 64);

  }

  if (op == 0x64737472) { ;; remove plugin by its request

    plugins~dict_delete?(8 + 256, wc_n_address);
    var ds = get_data().begin_parse().first_bits(32 + 32 + 256);
    set_data(begin_cell().store_slice(ds).store_dict(plugins).end_cell());
    ;; return coins only if bounce expected
    if (flags & 2) {
      msg = msg.store_uint(0x18, 6)
               .store_slice(s_addr)
               .store_grams(0)
               .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
