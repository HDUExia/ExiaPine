# ExiaPine

TradingView Pine Script，由 `PAQuant 自营图表 Fork v13.0` 重命名而来。

## 脚本说明

- 主图叠加显示日/周/月级别的开盘、最高、最低、收盘关键位。
- 支持夜盘高低点、历史最高价（ATH）。
- 成交量分布支持三种可勾选区间：昨日开盘 → 当前 K 线、今日开盘 → 当前 K 线、任意时间 → 当前 K 线；每个区间输出 POC、70% VAH、70% VAL，并以中断虚线形式绘制在每根 K 线上。POC 为黄色，VAH 为绿色，VAL 为红色。
- 最新 K 线右侧会显示 `昨开POC`、`昨开VAH`、`昨开VAL` 等中文价格标签。
- 支持 PAIO 的 Gap Series：Open、Wick、Body 三类缺口；缺口被价格回补时，已回补部分会用独立颜色标识。
- 提供本周期、15 分钟、60 分钟三档 EMA20 跨周期叠加。
- 副图显示 Standard ATR、Bull ATR、Bear ATR。
- 默认时区为 `America/New_York`，可在设置面板中切换。

## 脚本内容

```pine
//@version=6
indicator("ExiaPine", overlay = true, max_boxes_count = 500, max_lines_count = 500, max_labels_count = 500)

// ==========================================
// Forked from: PAQuant 自营图表 (时间戳无缝对齐·绝不压扁·夜盘美POC完整版) v12.0
// EMA 模块移植自: Price Action All In One (fengyangsi)
// ==========================================

// ==========================================
// 1. 控制面板
// ==========================================
// --- 时区与环境设置 ---
tz_option  = input.string("America/New_York", title="时间段基准时区", options=["America/New_York", "Asia/Shanghai", "UTC", "交易所默认(Exchange)"], group="基础与时区设置", tooltip="美股/外盘期货请选 America/New_York；国内期货请选 Asia/Shanghai")
tz_string  = tz_option == "交易所默认(Exchange)" ? syminfo.timezone : tz_option

show_d   = input.bool(true,  title="显示 日线级 水平线 (D)", group="常规水平线设置")
show_w   = input.bool(true,  title="显示 周线级 水平线 (W)", group="常规水平线设置")
show_m   = input.bool(true,  title="显示 月线级 水平线 (M)", group="常规水平线设置")
show_ath = input.bool(false, title="★ 显示 历史最高价 (ATH)", group="常规水平线设置")

// --- 夜盘设置 (默认按美东时间 18:00-09:30 为隔夜盘) ---
show_night = input.bool(true, title="显示 夜盘 高/低点", group="夜盘设置")
night_sess = input.session("1800-0930:1234567", title="夜盘时间段 (按上述时区)", group="夜盘设置", tooltip="美盘期货(NQ/ES): NY时区设 1800-0930；国内期货: BJ时区设 2100-0230")

// --- 成交量分布设置 (Volume Profile) ---
show_vp_prev   = input.bool(true,  title="显示 昨日开盘 → 当前 K 线", group="成交量分布设置")
show_vp_today  = input.bool(true,  title="显示 今日开盘 → 当前 K 线", group="成交量分布设置")
show_vp_custom = input.bool(false, title="显示 任意 K 线 → 当前 K 线", group="成交量分布设置")
vp_day_sess    = input.session("0930-1600:1234567", title="日盘开盘时段 (识别今日/昨日开盘)", group="成交量分布设置", tooltip="美盘期货(NQ/ES): NY时区设 0930-1600；若按BJ时区设 2130-0400")
vp_custom_time = input.time(timestamp("2026-10-01T09:30:00-05:00"), title="任意 K 线起始时间", group="成交量分布设置", tooltip="选择具体的日期和时间；该时间点及之后的 K 线会纳入计算")
poc_ticks      = input.int(5, title="POC 价格精度 (Ticks/步长)", minval=1, group="成交量分布设置")
vp_va_pct      = input.float(70.0, title="Value Area 百分比 (%)", minval=1, maxval=100, step=1, group="成交量分布设置") / 100.0

// --- 跨周期 EMA 核心参数 (PAIO 高周期 EMA20 移植) ---
show_emas    = input.bool(true,  title="显示 跨周期 EMA", group="EMA 设置")

// 1st EMA (当前周期)
ma_1_use =       input.bool(defval = true, title = '本周期 EMA20', inline = 'ma_1_1', group = 'EMA 设置')
ma_1_length =    input.int(defval = 20, minval = 0, title = 'Length', inline = 'ma_1_1', group = 'EMA 设置', display = display.none)
ma_1_offset =    input.int(defval = 0, title = 'Offset', inline = 'ma_1_1', group = 'EMA 设置', display = display.none)
ma_1_timeframe = input.timeframe(defval = '', title = 'Timeframe', inline = 'ma_1_2', group = 'EMA 设置', display = display.none)
ma_1_source =    input.source(defval = close, title = 'Source', inline = 'ma_1_2', group = 'EMA 设置', display = display.none)

// 2nd EMA (15M)
ma_2_use =       input.bool(defval = true, title = '15M EMA20', inline = 'ma_2_1', group = 'EMA 设置')
ma_2_length =    input.int(defval = 20, minval = 0, title = 'Length', inline = 'ma_2_1', group = 'EMA 设置', display = display.none)
ma_2_offset =    input.int(defval = 0, title = 'Offset', inline = 'ma_2_1', group = 'EMA 设置', display = display.none)
ma_2_timeframe = input.timeframe(defval = '15', title = 'Timeframe', inline = 'ma_2_2', group = 'EMA 设置', display = display.none)
ma_2_source =    input.source(defval = close, title = 'Source', inline = 'ma_2_2', group = 'EMA 设置', display = display.none)

// 3rd EMA (60M)
ma_3_use =       input.bool(defval = true, title = '60M EMA20', inline = 'ma_3_1', group = 'EMA 设置')
ma_3_length =    input.int(defval = 20, minval = 0, title = 'Length', inline = 'ma_3_1', group = 'EMA 设置', display = display.none)
ma_3_offset =    input.int(defval = 0, title = 'Offset', inline = 'ma_3_1', group = 'EMA 设置', display = display.none)
ma_3_timeframe = input.timeframe(defval = '60', title = 'Timeframe', inline = 'ma_3_2', group = 'EMA 设置', display = display.none)
ma_3_source =    input.source(defval = close, title = 'Source', inline = 'ma_3_2', group = 'EMA 设置', display = display.none)

// --- ATR 参数 ---
show_atr    = input.bool(true,  title="显示多空 ATR 副图", group="ATR 设置")
atrLength   = input.int(14, title="ATR 统计周期", minval=1, group="ATR 设置")

// --- 缺口设置 (Gap Series，移植自 PAIO) ---
gs_use            = input.bool(true, title="显示缺口系列", group="缺口设置")
gs_open_use       = input.bool(true, title="Open 缺口", inline="gs_open_inline", group="缺口设置")
gs_open_u_color   = input.color(color.new(#089981, 40), title="↑", inline="gs_open_inline", group="缺口设置")
gs_open_d_color   = input.color(color.new(#F23645, 40), title="↓", inline="gs_open_inline", group="缺口设置")
gs_wick_use       = input.bool(true, title="Wick 缺口", inline="gs_wick_inline", group="缺口设置")
gs_wick_u_color   = input.color(color.new(#089981, 60), title="↑", inline="gs_wick_inline", group="缺口设置")
gs_wick_d_color   = input.color(color.new(#F23645, 60), title="↓", inline="gs_wick_inline", group="缺口设置")
gs_wick_except_open = input.bool(true, title="排除 Open", inline="gs_wick_inline", group="缺口设置")
gs_body_use       = input.bool(true, title="Body 缺口", inline="gs_body_inline", group="缺口设置")
gs_body_u_color   = input.color(color.new(#089981, 80), title="↑", inline="gs_body_inline", group="缺口设置")
gs_body_d_color   = input.color(color.new(#F23645, 80), title="↓", inline="gs_body_inline", group="缺口设置")
gs_body_except_open = input.bool(true, title="排除 Open", inline="gs_body_inline", group="缺口设置")
gs_body_except_wick = input.bool(true, title="排除 Wick", inline="gs_body_inline", group="缺口设置")
gs_fill_color     = input.color(color.new(color.yellow, 30), title="回补颜色", group="缺口设置", tooltip="缺口被价格回补后，已回补部分使用该颜色标识")

// ==========================================
// 2. 局部渲染函数
// ==========================================
render_level(l_id, lbl_id, val, col, width, txt, is_enabled) =>
    if is_enabled and not na(val) and barstate.islast
        line.set_xy1(l_id, bar_index - 30, val)
        line.set_xy2(l_id, bar_index + 12, val)
        line.set_extend(l_id, extend.none)
        line.set_color(l_id, col)
        line.set_width(l_id, width)
        line.set_style(l_id, line.style_solid)

        label.set_xy(lbl_id, bar_index + 14, val)
        label.set_text(lbl_id, txt + " " + str.tostring(val, format.mintick))
        label.set_textcolor(lbl_id, col)
        label.set_style(lbl_id, label.style_none)
        label.set_textalign(lbl_id, text.align_left)
    else
        line.set_color(l_id, color.new(color.white, 100))
        label.set_textcolor(lbl_id, color.new(color.white, 100))

// 仅用于成交量分布折线：在最新 K 线右侧放置中文价格标签。
render_vp_label(lbl_id, val, col, txt, is_enabled, x_offset) =>
    if is_enabled and not na(val) and barstate.islast
        label.set_xy(lbl_id, bar_index + x_offset, val)
        label.set_text(lbl_id, txt + " " + str.tostring(val, format.mintick))
        label.set_textcolor(lbl_id, col)
        label.set_style(lbl_id, label.style_none)
        label.set_textalign(lbl_id, text.align_left)
    else
        label.set_textcolor(lbl_id, color.new(color.white, 100))

// 将一个 5 分钟 K 线的 OHLC 按 price bucket 累加到成交量分布数组。
// 每个 K 线拆成 O/H/L/C 四个采样点，各点分配 volume / 4。
f_add_volume_profile_bar(_prices, _vols, _step) =>
    float _v_part = volume / 4.0
    float _p_o = math.round(open  / _step) * _step
    float _p_h = math.round(high  / _step) * _step
    float _p_l = math.round(low   / _step) * _step
    float _p_c = math.round(close / _step) * _step

    array<float> _pts = array.new_float(4)
    array.set(_pts, 0, _p_o)
    array.set(_pts, 1, _p_h)
    array.set(_pts, 2, _p_l)
    array.set(_pts, 3, _p_c)

    for _i = 0 to 3
        float _pt = array.get(_pts, _i)
        int _idx  = array.indexof(_prices, _pt)
        if _idx != -1
            array.set(_vols, _idx, array.get(_vols, _idx) + _v_part)
        else
            array.push(_prices, _pt)
            array.push(_vols, _v_part)

// 返回成交量分布中的 POC、VAH、VAL。
// 算法：先取成交量最大的价格层级作为 POC，再围绕 POC 向两侧扩展，
// 每次选择相邻价格层级中成交量更大的一侧，直到覆盖指定百分比的成交量。
f_profile_levels(_prices, _vols, _va_pct) =>
    float _poc = na
    float _vah = na
    float _val = na

    if array.size(_vols) > 0
        int _poc_idx = array.indexof(_vols, array.max(_vols))
        if _poc_idx != -1
            _poc := array.get(_prices, _poc_idx)

            array<int> _sorted = array.sort_indices(_prices, order.ascending)
            int _poc_pos = array.indexof(_sorted, _poc_idx)
            if _poc_pos != -1
                float _total_vol = array.sum(_vols)
                float _target_vol = _total_vol * _va_pct

                int _lo = _poc_pos
                int _hi = _poc_pos
                float _acc_vol = array.get(_vols, array.get(_sorted, _poc_pos))

                while _acc_vol < _target_vol and (_lo > 0 or _hi < array.size(_sorted) - 1)
                    int _next_lo = _lo > 0 ? array.get(_sorted, _lo - 1) : -1
                    int _next_hi = _hi < array.size(_sorted) - 1 ? array.get(_sorted, _hi + 1) : -1

                    if _lo <= 0
                        _hi += 1
                        _acc_vol += array.get(_vols, _next_hi)
                    else if _hi >= array.size(_sorted) - 1
                        _lo -= 1
                        _acc_vol += array.get(_vols, _next_lo)
                    else
                        float _vol_lo = array.get(_vols, _next_lo)
                        float _vol_hi = array.get(_vols, _next_hi)
                        if _vol_lo >= _vol_hi
                            _lo -= 1
                            _acc_vol += _vol_lo
                        else
                            _hi += 1
                            _acc_vol += _vol_hi

                _val := array.get(_prices, array.get(_sorted, _lo))
                _vah := array.get(_prices, array.get(_sorted, _hi))

    [_poc, _vah, _val]

// ==========================================
// 3. 全时段 (ETH) 跨域数据提取算法
// ==========================================
ext_symbol = ticker.new(syminfo.prefix, syminfo.ticker, session.extended)

// --- A. 常规 D/W/M 级高低开收 (直接取上一周期 [1]，彻底解决回放失效) ---
[y_o, y_h, y_l, y_c] = request.security(ext_symbol, "D", [open[1], high[1], low[1], close[1]], gaps = barmerge.gaps_off)
[w_o, w_h, w_l, w_c] = request.security(ext_symbol, "W", [open[1], high[1], low[1], close[1]], gaps = barmerge.gaps_off)
[m_o, m_h, m_l, m_c] = request.security(ext_symbol, "M", [open[1], high[1], low[1], close[1]], gaps = barmerge.gaps_off)

// --- B. 夜盘高低点锁存（基于 5 分钟 ETH 数据流，避开 RTH 裁剪） ---
f_calc_night_levels(_night_sess, _tz) =>
    string _calc_tf = "5"

    _in_night = not na(time(_calc_tf, _night_sess, _tz))
    var float _live_nh = na, var float _live_nl = na
    var float _last_completed_nh = na, var float _last_completed_nl = na

    if _in_night
        if not _in_night[1] // 新一轮夜盘开启
            if not na(_live_nh)
                _last_completed_nh := _live_nh
                _last_completed_nl := _live_nl
            _live_nh := high
            _live_nl := low
        else
            _live_nh := math.max(nz(_live_nh, high), high)
            _live_nl := math.min(nz(_live_nl, low), low)
    else if _in_night[1] // 夜盘刚好结束（09:30），锁存最终夜高夜低
        _last_completed_nh := _live_nh
        _last_completed_nl := _live_nl

    float _out_nh = _in_night ? _live_nh : _last_completed_nh
    float _out_nl = _in_night ? _live_nl : _last_completed_nl

    [_out_nh, _out_nl]

// --- C. 三种成交量分布区间（POC / VAH / VAL） ---
f_calc_volume_profiles(_day_sess, _custom_start, _poc_ticks, _va_pct, _tz) =>
    string _calc_tf = "5"

    _in_day = not na(time(_calc_tf, _day_sess, _tz))
    _is_new_day = _in_day and not _in_day[1]

    var array<float> _prev_prices = array.new_float(0)
    var array<float> _prev_vols   = array.new_float(0)
    var array<float> _today_prices = array.new_float(0)
    var array<float> _today_vols   = array.new_float(0)
    var array<float> _custom_prices = array.new_float(0)
    var array<float> _custom_vols   = array.new_float(0)
    var bool _day_seen = false

    if _is_new_day
        // 新日盘开盘：把已完成的本日数组转存为“昨日开盘”数组，再重置“今日开盘”数组
        _prev_prices := array.copy(_today_prices)
        _prev_vols   := array.copy(_today_vols)
        _today_prices := array.new_float(0)
        _today_vols   := array.new_float(0)
        _day_seen := true

    float _step = syminfo.mintick * _poc_ticks

    if _day_seen
        f_add_volume_profile_bar(_today_prices, _today_vols, _step)
        f_add_volume_profile_bar(_prev_prices, _prev_vols, _step)

    if time >= _custom_start
        f_add_volume_profile_bar(_custom_prices, _custom_vols, _step)

    [_prev_poc, _prev_vah, _prev_val]       = f_profile_levels(_prev_prices, _prev_vols, _va_pct)
    [_today_poc, _today_vah, _today_val]    = f_profile_levels(_today_prices, _today_vols, _va_pct)
    [_custom_poc, _custom_vah, _custom_val] = f_profile_levels(_custom_prices, _custom_vols, _va_pct)

    [_prev_poc, _prev_vah, _prev_val, _today_poc, _today_vah, _today_val, _custom_poc, _custom_vah, _custom_val]

// 跨域使用 "5" 分钟全时段 (ETH) 抓取数据
[night_h, night_l] = request.security(ext_symbol, "5", f_calc_night_levels(night_sess, tz_string), gaps = barmerge.gaps_off)

[vp_prev_poc, vp_prev_vah, vp_prev_val, vp_today_poc, vp_today_vah, vp_today_val, vp_custom_poc, vp_custom_vah, vp_custom_val] = request.security(ext_symbol, "5", f_calc_volume_profiles(vp_day_sess, vp_custom_time, poc_ticks, vp_va_pct, tz_string), gaps = barmerge.gaps_off)

// 历史最高价
ath = ta.highest(high, bar_index + 1)

// ==========================================
// 4. 图形对象初始化与渲染
// ==========================================
var l_yo = line.new(na,na,na,na), var lbl_yo = label.new(na,na,"")
var l_yh = line.new(na,na,na,na), var lbl_yh = label.new(na,na,"")
var l_yl = line.new(na,na,na,na), var lbl_yl = label.new(na,na,"")
var l_yc = line.new(na,na,na,na), var lbl_yc = label.new(na,na,"")

var l_wo = line.new(na,na,na,na), var lbl_wo = label.new(na,na,"")
var l_wh = line.new(na,na,na,na), var lbl_wh = label.new(na,na,"")
var l_wl = line.new(na,na,na,na), var lbl_wl = label.new(na,na,"")
var l_wc = line.new(na,na,na,na), var lbl_wc = label.new(na,na,"")

var l_mo = line.new(na,na,na,na), var lbl_mo = label.new(na,na,"")
var l_mh = line.new(na,na,na,na), var lbl_mh = label.new(na,na,"")
var l_ml = line.new(na,na,na,na), var lbl_ml = label.new(na,na,"")
var l_mc = line.new(na,na,na,na), var lbl_mc = label.new(na,na,"")

var l_nh  = line.new(na,na,na,na), var lbl_nh  = label.new(na,na,"")
var l_nl  = line.new(na,na,na,na), var lbl_nl  = label.new(na,na,"")
var lbl_vp_prev_poc = label.new(na,na,"")
var lbl_vp_prev_vah = label.new(na,na,"")
var lbl_vp_prev_val = label.new(na,na,"")
var lbl_vp_today_poc = label.new(na,na,"")
var lbl_vp_today_vah = label.new(na,na,"")
var lbl_vp_today_val = label.new(na,na,"")
var lbl_vp_custom_poc = label.new(na,na,"")
var lbl_vp_custom_vah = label.new(na,na,"")
var lbl_vp_custom_val = label.new(na,na,"")
var l_ath = line.new(na,na,na,na), var lbl_ath = label.new(na,na,"")

render_level(l_yo, lbl_yo, y_o, color.new(#b2b5be, 40), 1, "日开", show_d)
render_level(l_yh, lbl_yh, y_h, color.new(#b2b5be, 40), 1, "日高", show_d)
render_level(l_yl, lbl_yl, y_l, color.new(#b2b5be, 40), 1, "日低", show_d)
render_level(l_yc, lbl_yc, y_c, color.new(#b2b5be, 40), 1, "日收", show_d)

render_level(l_wo, lbl_wo, w_o, color.new(#b2b5be, 0),  2, "周开", show_w)
render_level(l_wh, lbl_wh, w_h, color.new(#b2b5be, 0),  2, "周高", show_w)
render_level(l_wl, lbl_wl, w_l, color.new(#b2b5be, 0),  2, "周低", show_w)
render_level(l_wc, lbl_wc, w_c, color.new(#b2b5be, 0),  2, "周收", show_w)

render_level(l_mo, lbl_mo, m_o, color.new(#b2b5be, 0),  2, "月开", show_m)
render_level(l_mh, lbl_mh, m_h, color.new(#b2b5be, 0),  2, "月高", show_m)
render_level(l_ml, lbl_ml, m_l, color.new(#b2b5be, 0),  2, "月低", show_m)
render_level(l_mc, lbl_mc, m_c, color.new(#b2b5be, 0),  2, "月收", show_m)

render_level(l_nh,  lbl_nh,  night_h,     color.new(#b2b5be, 0), 1, "夜高", show_night)
render_level(l_nl,  lbl_nl,  night_l,     color.new(#b2b5be, 0), 1, "夜低", show_night)
plot(series = show_vp_prev and bar_index % 4 != 3 ? vp_prev_poc : na, title = "昨开POC", color = color.new(#ffd600, 0), linewidth = 2, style = plot.style_linebr)
plot(series = show_vp_prev and bar_index % 4 != 3 ? vp_prev_vah : na, title = "昨开VAH", color = color.new(#00e676, 0), linewidth = 2, style = plot.style_linebr)
plot(series = show_vp_prev and bar_index % 4 != 3 ? vp_prev_val : na, title = "昨开VAL", color = color.new(#ff5252, 0), linewidth = 2, style = plot.style_linebr)
plot(series = show_vp_today and bar_index % 4 != 3 ? vp_today_poc : na, title = "今开POC", color = color.new(#ffd600, 0), linewidth = 2, style = plot.style_linebr)
plot(series = show_vp_today and bar_index % 4 != 3 ? vp_today_vah : na, title = "今开VAH", color = color.new(#00e676, 0), linewidth = 2, style = plot.style_linebr)
plot(series = show_vp_today and bar_index % 4 != 3 ? vp_today_val : na, title = "今开VAL", color = color.new(#ff5252, 0), linewidth = 2, style = plot.style_linebr)
plot(series = show_vp_custom and bar_index % 4 != 3 ? vp_custom_poc : na, title = "自定义POC", color = color.new(#ffd600, 0), linewidth = 2, style = plot.style_linebr)
plot(series = show_vp_custom and bar_index % 4 != 3 ? vp_custom_vah : na, title = "自定义VAH", color = color.new(#00e676, 0), linewidth = 2, style = plot.style_linebr)
plot(series = show_vp_custom and bar_index % 4 != 3 ? vp_custom_val : na, title = "自定义VAL", color = color.new(#ff5252, 0), linewidth = 2, style = plot.style_linebr)

render_vp_label(lbl_vp_prev_poc, vp_prev_poc, color.new(#ffd600, 0), "昨开POC", show_vp_prev, 1)
render_vp_label(lbl_vp_prev_vah, vp_prev_vah, color.new(#00e676, 0), "昨开VAH", show_vp_prev, 3)
render_vp_label(lbl_vp_prev_val, vp_prev_val, color.new(#ff5252, 0), "昨开VAL", show_vp_prev, 5)
render_vp_label(lbl_vp_today_poc, vp_today_poc, color.new(#ffd600, 0), "今开POC", show_vp_today, 1)
render_vp_label(lbl_vp_today_vah, vp_today_vah, color.new(#00e676, 0), "今开VAH", show_vp_today, 3)
render_vp_label(lbl_vp_today_val, vp_today_val, color.new(#ff5252, 0), "今开VAL", show_vp_today, 5)
render_vp_label(lbl_vp_custom_poc, vp_custom_poc, color.new(#ffd600, 0), "自定义POC", show_vp_custom, 1)
render_vp_label(lbl_vp_custom_vah, vp_custom_vah, color.new(#00e676, 0), "自定义VAH", show_vp_custom, 3)
render_vp_label(lbl_vp_custom_val, vp_custom_val, color.new(#ff5252, 0), "自定义VAL", show_vp_custom, 5)

render_level(l_ath, lbl_ath, ath, color.new(#b2b5be, 0), 3, "★ ATH", show_ath)

// ==========================================
// 5. 主图模块：跨周期 EMA (PAIO 移植，高周期 EMA20 落到低周期)
// ==========================================
ma_1_request = request.security(symbol = syminfo.tickerid, timeframe = ma_1_timeframe, expression = ta.ema(ma_1_source, ma_1_length), gaps = barmerge.gaps_on)
var float ma_1_last = na
var float ma_1_value = na
if na(ma_1_request)
    ma_1_value := (ma_1_source - ma_1_last) * (2 / (ma_1_length + 1)) + ma_1_last
else
    ma_1_value := ma_1_request
    ma_1_last := ma_1_value
plot(series = ma_1_use and show_emas ? ma_1_value : na, title = '本周期 EMA20', color = color.new(#9daaff, 30), linewidth = 1, style = plot.style_line, offset = ma_1_offset)

ma_2_request = request.security(symbol = syminfo.tickerid, timeframe = ma_2_timeframe, expression = ta.ema(ma_2_source, ma_2_length), gaps = barmerge.gaps_on)
var float ma_2_last = na
var float ma_2_value = na
if na(ma_2_request)
    ma_2_value := (ma_2_source - ma_2_last) * (2 / (ma_2_length + 1)) + ma_2_last
else
    ma_2_value := ma_2_request
    ma_2_last := ma_2_value
plot(series = ma_2_use and show_emas ? ma_2_value : na, title = '15M EMA20', color = color.new(#ff7043, 0), linewidth = 2, style = plot.style_line, offset = ma_2_offset)

ma_3_request = request.security(symbol = syminfo.tickerid, timeframe = ma_3_timeframe, expression = ta.ema(ma_3_source, ma_3_length), gaps = barmerge.gaps_on)
var float ma_3_last = na
var float ma_3_value = na
if na(ma_3_request)
    ma_3_value := (ma_3_source - ma_3_last) * (2 / (ma_3_length + 1)) + ma_3_last
else
    ma_3_value := ma_3_request
    ma_3_last := ma_3_value
plot(series = ma_3_use and show_emas ? ma_3_value : na, title = '60M EMA20', color = color.new(#d500f9, 0), linewidth = 2, style = plot.style_line, offset = ma_3_offset)

// ==========================================
// 6. 副图模块：多空非对称 ATR
// ==========================================
standardTR = ta.tr(true)
bullTR     = math.max(0.0, high - high[1])
bearTR     = math.max(0.0, low[1] - low)

final_ATR     = ta.rma(standardTR, atrLength)
final_BullATR = ta.rma(bullTR, atrLength)
final_BearATR = ta.rma(bearTR, atrLength)

plot(show_atr ? final_ATR : na,     title="Standard ATR", color=color.new(color.gray, 30), linewidth=2, force_overlay=false)
plot(show_atr ? final_BullATR : na, title="Bull ATR",     color=color.new(#00e676, 0),    linewidth=1, force_overlay=false)
plot(show_atr ? final_BearATR : na, title="Bear ATR",     color=color.new(#ff5252, 0),    linewidth=1, force_overlay=false)

// ==========================================
// 7. 缺口系列 (Gap Series，移植自 PAIO + 回补变色)
// ==========================================
var array<box> _gap_fill_boxes = array.new<box>()
var array<box> _gap_boxes = array.new<box>()
var array<int> _gap_dirs = array.new<int>()
var array<float> _gap_top_limits = array.new<float>()
var array<float> _gap_bottom_limits = array.new<float>()

f_add_gap(_boxes, _fill_boxes, _dirs, _top_limits, _bottom_limits, left, top, right, bottom, bgcolor, fillcolor, dir, entry_top, entry_bottom) =>
    box _orig_box = box.new(left, top, right, bottom, border_width = 0, bgcolor = bgcolor)
    box _fill_box = box.new(left, entry_top, right, entry_bottom, border_width = 0, bgcolor = fillcolor)
    array.push(_boxes, _orig_box)
    array.push(_fill_boxes, _fill_box)
    array.push(_dirs, dir)
    array.push(_top_limits, top)
    array.push(_bottom_limits, bottom)

bool gs_open_u_alert = false
bool gs_open_d_alert = false
if low > high[1]
    gs_open_u_alert := true
    if gs_use and gs_open_use
        f_add_gap(_gap_boxes, _gap_fill_boxes, _gap_dirs, _gap_top_limits, _gap_bottom_limits, bar_index[1], low, bar_index, high[1], gs_open_u_color, gs_fill_color, 1, low, low)
else if high < low[1]
    gs_open_d_alert := true
    if gs_use and gs_open_use
        f_add_gap(_gap_boxes, _gap_fill_boxes, _gap_dirs, _gap_top_limits, _gap_bottom_limits, bar_index[1], low[1], bar_index, high, gs_open_d_color, gs_fill_color, -1, high, high)

bool gs_wick_u_alert = false
bool gs_wick_d_alert = false
if low > high[2]
    gs_wick_u_alert := true
    if gs_use and gs_wick_use and ((gs_wick_except_open and not gs_open_u_alert and not gs_open_u_alert[1]) or not gs_wick_except_open)
        f_add_gap(_gap_boxes, _gap_fill_boxes, _gap_dirs, _gap_top_limits, _gap_bottom_limits, bar_index[2], low, bar_index, high[2], gs_wick_u_color, gs_fill_color, 1, low, low)
else if high < low[2]
    gs_wick_d_alert := true
    if gs_use and gs_wick_use and ((gs_wick_except_open and not gs_open_d_alert and not gs_open_d_alert[1]) or not gs_wick_except_open)
        f_add_gap(_gap_boxes, _gap_fill_boxes, _gap_dirs, _gap_top_limits, _gap_bottom_limits, bar_index[2], low[2], bar_index, high, gs_wick_d_color, gs_fill_color, -1, high, high)

float gs_body_high = math.max(open, close)
float gs_body_low  = math.min(open, close)
bool gs_body_u_alert = false
bool gs_body_d_alert = false
if gs_body_low > gs_body_high[2]
    gs_body_u_alert := true
    if gs_use and gs_body_use and ((gs_body_except_open and not gs_open_u_alert and not gs_open_u_alert[1]) or not gs_body_except_open) and ((gs_body_except_wick and not gs_wick_u_alert) or not gs_body_except_wick)
        f_add_gap(_gap_boxes, _gap_fill_boxes, _gap_dirs, _gap_top_limits, _gap_bottom_limits, bar_index[2], gs_body_high[2], bar_index, gs_body_low, gs_body_u_color, gs_fill_color, 1, gs_body_low, gs_body_low)
else if gs_body_high < gs_body_low[2]
    gs_body_d_alert := true
    if gs_use and gs_body_use and ((gs_body_except_open and not gs_wick_d_alert and not gs_open_d_alert) or not gs_body_except_open) and ((gs_body_except_wick and not gs_open_d_alert[1]) or not gs_body_except_wick)
        f_add_gap(_gap_boxes, _gap_fill_boxes, _gap_dirs, _gap_top_limits, _gap_bottom_limits, bar_index[2], gs_body_low[2], bar_index, gs_body_high, gs_body_d_color, gs_fill_color, -1, gs_body_high, gs_body_high)

if array.size(_gap_fill_boxes) > 0
    int _gi = 0
    while _gi < array.size(_gap_fill_boxes)
        box _orig_box = array.get(_gap_boxes, _gi)
        box _fill_box = array.get(_gap_fill_boxes, _gi)
        int _dir = array.get(_gap_dirs, _gi)
        float _top_limit = array.get(_gap_top_limits, _gi)
        float _bottom_limit = array.get(_gap_bottom_limits, _gi)
        bool _fully_filled = false

        if _dir > 0
            float _filled_bottom = box.get_bottom(_fill_box)
            _filled_bottom := math.min(_filled_bottom, low)
            _filled_bottom := math.max(_filled_bottom, _bottom_limit)
            box.set_bottom(_fill_box, _filled_bottom)
            _fully_filled := _filled_bottom <= _bottom_limit
        else
            float _filled_top = box.get_top(_fill_box)
            _filled_top := math.max(_filled_top, high)
            _filled_top := math.min(_filled_top, _top_limit)
            box.set_top(_fill_box, _filled_top)
            _fully_filled := _filled_top >= _top_limit

        if _fully_filled
            box.delete(_orig_box)
            box.delete(_fill_box)
            array.remove(_gap_boxes, _gi)
            array.remove(_gap_fill_boxes, _gi)
            array.remove(_gap_dirs, _gi)
            array.remove(_gap_top_limits, _gi)
            array.remove(_gap_bottom_limits, _gi)
        else
            _gi += 1
```
