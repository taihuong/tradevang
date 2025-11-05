# tradevang import MetaTrader5 as mt5
import pandas as pd
import numpy as np
import time
from datetime import datetime

# ================== CÀI ĐẶT ==================
SYMBOL = "XAUUSD"
VOLUME = 0.01  # Khối lượng giao dịch (lot)
DEVIATION = 10
MAGIC_NUMBER = 123456
TIMEFRAME = mt5.TIMEFRAME_M15  # Khung thời gian 15 phút
MA_FAST = 9    # Đường MA nhanh
MA_SLOW = 21   # Đường MA chậm

# ================== KẾT NỐI MT5 ==================
def connect_mt5():
    if not mt5.initialize():
        print("Không thể kết nối MT5!")
        return False
    print(f"Đã kết nối MT5 - Phiên bản: {mt5.version()}")
    return True

# ================== LẤY DỮ LIỆU GIÁ ==================
def get_data(symbol, timeframe, n=100):
    rates = mt5.copy_rates_from_pos(symbol, timeframe, 0, n)
    df = pd.DataFrame(rates)
    df['time'] = pd.to_datetime(df['time'], unit='s')
    return df

# ================== TÍNH TOÁN CHỈ BÁO ==================
def calculate_signals(df):
    df['ma_fast'] = df['close'].rolling(window=MA_FAST).mean()
    df['ma_slow'] = df['close'].rolling(window=MA_SLOW).mean()
    
    # Tín hiệu mua: MA nhanh cắt lên MA chậm
    # Tín hiệu bán: MA nhanh cắt xuống MA chậm
    df['signal'] = 0
    df['signal'][MA_FAST:] = np.where(
        df['ma_fast'][MA_FAST:] > df['ma_slow'][MA_FAST:], 1, 0)
    df['position'] = df['signal'].diff()
    
    return df

# ================== GỬI LỆNH GIAO DỊCH ==================
def send_order(symbol, order_type, volume, price, sl=None, tp=None):
    request = {
        "action": mt5.TRADE_ACTION_DEAL,
        "symbol": symbol,
        "volume": volume,
        "type": order_type,
        "price": price,
        "sl": sl,
        "tp": tp,
        "deviation": DEVIATION,
        "magic": MAGIC_NUMBER,
        "comment": "Auto Gold Bot",
        "type_time": mt5.ORDER_TIME_GTC,
        "type_filling": mt5.ORDER_FILLING_IOC,
    }
    
    result = mt5.order_send(request)
    if result.retcode != mt5.TRADE_RETCODE_DONE:
        print(f"Lệnh thất bại: {result.comment}")
        return False
    else:
        print(f"Đã mở lệnh {order_type}: {volume} lot tại {price}")
        return True

# ================== KIỂM TRA VỊ THẾ HIỆN TẠI ==================
def get_position(symbol):
    positions = mt5.positions_get(symbol=symbol)
    if positions is None or len(positions) == 0:
        return None
    return positions[0]

# ================== CHƯƠNG TRÌNH CHÍNH ==================
def main():
    if not connect_mt5():
        return
    
    # Kiểm tra symbol có sẵn không
    if not mt5.symbol_select(SYMBOL, True):
        print(f"Không tìm thấy {SYMBOL}")
        mt5.shutdown()
        return
    
    print(f"Bắt đầu giao dịch tự động {SYMBOL}...")
    
    while True:
        try:
            # Lấy dữ liệu
            df = get_data(SYMBOL, TIMEFRAME, MA_SLOW + 10)
            if len(df) < MA_SLOW:
                time.sleep(60)
                continue
            
            df = calculate_signals(df)
            latest = df.iloc[-1]
            prev = df.iloc[-2]
            
            current_position = get_position(SYMBOL)
            
            # Tín hiệu mua
            if prev['position'] == 1 and (current_position is None or current_position.type == mt5.POSITION_TYPE_SELL):
                print(f"[{datetime.now()}] Tín hiệu MUA tại {latest['close']}")
                mt5.Close(SYMBOL)  # Đóng lệnh bán nếu có
                send_order(SYMBOL, mt5.ORDER_TYPE_BUY, VOLUME, mt5.symbol_info_tick(SYMBOL).ask)
            
            # Tín hiệu bán
            elif prev['position'] == -1 and (current_position is None or current_position.type == mt5.POSITION_TYPE_BUY):
                print(f"[{datetime.now()}] Tín hiệu BÁN tại {latest['close']}")
                mt5.Close(SYMBOL)  # Đóng lệnh mua nếu có
                send_order(SYMBOL, mt5.ORDER_TYPE_SELL, VOLUME, mt5.symbol_info_tick(SYMBOL).bid)
            
            else:
                print(f"[{datetime.now()}] Đang chờ tín hiệu... Giá hiện tại: {latest['close']:.2f}")
            
            time.sleep(60)  # Chờ 1 phút
            
        except Exception as e:
            print(f"Lỗi: {e}")
            time.sleep(60)
    
    mt5.shutdown()

# ================== CHẠY BOT ==================
if __name__ == "__main__":
    main()
