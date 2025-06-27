# chi-bao-update-2
//@version=6
indicator("GCM Heikin Ashi with Pivots", shorttitle="GCM HAP", overlay=false, format=format.price, precision=2)

// --- Group 0: Wave Scale Conversion (Values are set here in "Inputs") ---
string g0 = "Wave Scale Conversion"
int oscLookback = input.int(50, title="Oscillator Lookback Period", minval=10, group=g0, tooltip="The number of bars to calculate the price range for normalization. A larger number creates a smoother, less volatile oscillator.")
float maxLevel = input.float(20.0, title="Oscillator Upper Limit", group=g0, inline="scale_limits", tooltip="The maximum value of the oscillator's scale.")
float minLevel = input.float(-20.0, title="Oscillator Lower Limit", group=g0, inline="scale_limits", tooltip="The minimum value of the oscillator's scale.")
float obLevel = input.float(12.0, "Overbought Level", group=g0, inline="levels")
float osLevel = input.float(-12.0, "Oversold Level", group=g0, inline="levels")

// --- Group 1: Pivot Detection Inputs ---
string g1 = "Pivot Detection"
int leftBars = input.int(10, title="Left Bars (Strength)", minval=1, group=g1)
int rightBars = input.int(5, title="Right Bars (Confirmation)", minval=1, tooltip="The number of bars to the right required to confirm a pivot. This introduces a delay. A label will appear 'Right Bars' after the pivot has formed.", group=g1)

// --- Group 2: Volume Spike Detection (VROC) ---
string g2 = "Volume Spike Detection (VROC)"
bool enableVroc = input.bool(true, title="Enable Volume Spike Highlighting", group=g2)
int vrocLength = input.int(14, title="VROC Length", minval=1, group=g2)
float vrocThreshold = input.float(50, title="VROC Threshold %", minval=0, group=g2, tooltip="Volume must increase by more than this percentage to be highlighted.")
color vrocBuyHighlightColor = input.color(#2dff00, "VROC Buy Spike Color", group=g2, inline="vroc_buy")
color vrocSellHighlightColor = input.color(#ff00e8, "VROC Sell Spike Color", group=g2, inline="vroc_sell")

// --- Group 3: Label Customization Inputs ---
string g3 = "Label Customization"
string label_text_mode = input.string("High & Low", title="Label Text Mode", options=["High & Low", "Buy & Sell"], group=g3)
color h_label_color = input.color(#801922, title="High Label Color", group=g3, inline="h_label")
color h_text_color = input.color(color.new(color.white, 0), title="Text Color", group=g3, inline="h_label")
string h_label_style_str = input.string("Box Down", title="High Label Style", options=["Box Down", "Circle", "Diamond", "Square"], group=g3)
color l_label_color = input.color(#1b5e20, title="Low Label Color", group=g3, inline="l_label")
color l_text_color = input.color(color.new(color.white, 0), title="Text Color", group=g3, inline="l_label")
string l_label_style_str = input.string("Box Up", title="Low Label Style", options=["Box Up", "Circle", "Diamond", "Square"], group=g3)
string label_size_str = input.string("Tiny", title="Label Size", options=["Tiny", "Small", "Normal", "Large", "Huge"], group=g3)

// --- Heikin Ashi Calculation ---
string ha_ticker = ticker.heikinashi(syminfo.tickerid)
[ha_open, ha_high, ha_low, ha_close] = request.security(ha_ticker, timeframe.period, [open, high, low, close], lookahead=barmerge.lookahead_off)

// --- Normalization & Scaling Logic ---
float highestHaInRange = ta.highest(ha_high, oscLookback)
float lowestHaInRange = ta.lowest(ha_low, oscLookback)
float priceRange = highestHaInRange - lowestHaInRange
priceRange := priceRange == 0 ? 1 : priceRange 

f_normalizeAndScale(price, min, pRange) =>
    normalizedValue = (price - min) / pRange
    scaledValue = (normalizedValue * (maxLevel - minLevel)) + minLevel
    scaledValue

float osc_open = f_normalizeAndScale(ha_open, lowestHaInRange, priceRange)
float osc_high = f_normalizeAndScale(ha_high, lowestHaInRange, priceRange)
float osc_low = f_normalizeAndScale(ha_low, lowestHaInRange, priceRange)
float osc_close = f_normalizeAndScale(ha_close, lowestHaInRange, priceRange)

// --- VROC Calculation ---
float volumeRoc = ta.roc(volume, vrocLength)
bool isVrocSpike = enableVroc and (volumeRoc > vrocThreshold)
bool isVrocBuySpike = isVrocSpike and (close > open)
bool isVrocSellSpike = isVrocSpike and (close < open)

// --- Determine Final Candle Color ---
color final_ha_color = isVrocBuySpike ? vrocBuyHighlightColor : isVrocSellSpike ? vrocSellHighlightColor : ha_close >= ha_open ? color.new(color.green, 0) : color.new(color.red, 0)

// --- Plotting Section ---
// All appearance controls (Color, Thickness, Line Style) are in the "Style" tab
plot_upper_limit = plot(maxLevel, "Upper Limit", color=color.new(color.red, 50))
plot_lower_limit = plot(minLevel, "Lower Limit", color=color.new(color.green, 50))
ob_plot = plot(obLevel, "Overbought", color=color.new(color.red, 50))
os_plot = plot(osLevel, "Oversold", color=color.new(color.green, 50))
plot(0, "Centerline", color=color.new(color.gray, 70))

fill(ob_plot, plot_upper_limit, color=color.new(color.red, 90), title="OB Zone")
fill(os_plot, plot_lower_limit, color=color.new(color.green, 90), title="OS Zone")

plotcandle(osc_open, osc_high, osc_low, osc_close, title="Heikin Ashi Wave Candles", color=final_ha_color, wickcolor=final_ha_color, bordercolor=final_ha_color)

// --- Pivot High/Low Logic ---
float pivotHighValue = ta.pivothigh(high, leftBars, rightBars)
float pivotLowValue = ta.pivotlow(low, leftBars, rightBars)

// --- Convert String Inputs to Pine Script Constants ---
string high_text_to_plot = label_text_mode == "High & Low" ? "H" : "S"
string low_text_to_plot  = label_text_mode == "High & Low" ? "L" : "B"

var string high_style = switch h_label_style_str
    "Circle"    => label.style_circle
    "Diamond"   => label.style_diamond
    "Square"    => label.style_square
    => label.style_label_down

var string low_style = switch l_label_style_str
    "Circle"    => label.style_circle
    "Diamond"   => label.style_diamond
    "Square"    => label.style_square
    => label.style_label_up

var string label_size = switch label_size_str
    "Small"     => size.small
    "Normal"    => size.normal
    "Large"     => size.large
    "Huge"      => size.huge
    => size.tiny

// --- Plot H/L Labels at Confirmed Pivots ---
if not na(pivotHighValue)
    float label_y_high = f_normalizeAndScale(ha_high[rightBars], lowestHaInRange[rightBars], priceRange[rightBars])
    label.new(bar_index[rightBars], label_y_high, text=high_text_to_plot, color=h_label_color, textcolor=h_text_color, style=high_style, yloc=yloc.price, size=label_size)
if not na(pivotLowValue)
    float label_y_low = f_normalizeAndScale(ha_low[rightBars], lowestHaInRange[rightBars], priceRange[rightBars])
    label.new(bar_index[rightBars], label_y_low, text=low_text_to_plot, color=l_label_color, textcolor=l_text_color, style=low_style, yloc=yloc.price, size=label_size)****
