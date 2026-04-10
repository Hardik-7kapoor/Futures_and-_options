# Futures_and-_options
In this i have made an algorithm which basically does help in api trading in indian Market 
from datetime import datetime, timedelta
import time
import pymongo
import pause , logging

from os import getenv
from os import kill
from os import getpid
# from dotenv import load_dotenv
from sys import argv

from strat_constants import Params
from strat_constants import R3
from strat_constants import Tick
from strat_constants import DbOrder
from strat_constants import DbOrder
from strat_constants import Strat
from strat_constants import Err
from strat_constants import CANDLE

from fenix import WeeklyExpiry
from fenix import Root
from fenix import Option
from fenix import OrderType
from fenix import mastertrust

from os import path
# from utils import file_path_locator
# from utils import log_setup

import traceback


def Trade_count():
    _id = str(params[R3._ID])
    
    prev_trade_cursor = historical_collec.find({
        "original_id": _id,
        "date": todays_date
    })

    prev_trade = list(prev_trade_cursor)

    if len(prev_trade) >= 2:
        logging.info("..................BYE BYE ITS END FOR THE DAY .................")
        kill(getpid(), 9)
    

def sec_position_creator():
    logging.info("SECOND TIME ENTRY  STARTED .......")
    _id = str(params[R3._ID])
    position_1 = positions_collec.find_one(
        {R3._ID: _id}
        # {R3.DATE: todays_date}, sort=[(R3._ID, pymongo.DESCENDING)]
    )

    if position_1:
        original_id = position_1['_id']
        positions_collec.delete_one({R3._ID: _id})
        position_1['original_id'] = original_id
        position_1.pop('_id', None)
        historical_collec.insert_one(position_1)
        time.sleep(15)
    fetching_positions()
    # # positions, _ = fetching_positions()
    # ce_buy = positions[R3.CE_buy]
    # pe_buy = positions[R3.PE_buy]
    logging.info("Globals refreshed with blank template for 2nd entry.")


def fetching_positions():
    _id = str(params[R3._ID])
    print("HII")
    fetch_positions = positions_collec.find_one(
        {R3._ID: _id}#, sort=[(R3._ID, pymongo.DESCENDING)]
    )
    if fetch_positions and fetch_positions[R3.DATE] != todays_date:
        sec_position_creator()

    if not fetch_positions:
        _id = str(params[R3._ID]) #str(time.perf_counter_ns())

        def_data = {
            R3._ID: _id,
            R3.DATE: todays_date,
            R3.POS: 0,
            R3.NF_PRICE: 0,
            R3.CE_LIST:0,
            R3.PE_LIST:0,
            R3.ATM: 0,
            # R3.NF_TIME: 0,
            # R3.INDEX_PERC: 0,
            R3.CE_buy: {

                R3.ID: 0,
                R3.TOKEN: 0,
                R3.PRICE: 0,
                R3.SYMBOL: 0,
                R3.ROOT: 0,
                R3.TOKEN: 0,
                R3.BROKER_DATA: 0,
                R3.LOTS: 0,
                R3.EN_ID: 0,
                R3.EN_PRICE: 0,
                R3.EN_TIME: 0,
                R3.SL_ID: 0,
                R3.TARGET: 0,
                R3.STOPLOSS_1:0,
                R3.TRIGGER_1: 0,
                R3.STOPLOSS_2:0,
                R3.TRIGGER_2: 0,
                R3.EX_PRICE: 0,
                R3.EX_TIME: 0,
                R3.EX_TYPE: 0,
            },
            R3.PE_buy: {
                R3.ID: 0,
                R3.TOKEN: 0,
                R3.PRICE: 0,
                R3.SYMBOL: 0,
                R3.ROOT: 0,
                R3.TOKEN: 0,
                R3.BROKER_DATA: 0,
                R3.LOTS: 0,
                R3.EN_ID: 0,
                R3.EN_PRICE: 0,
                R3.EN_TIME: 0,
                R3.SL_ID: 0,
                R3.STOPLOSS_1:0,
                R3.TRIGGER_1: 0,
                R3.STOPLOSS_2:0,
                R3.TRIGGER_2: 0,
                R3.TARGET: 0,
                R3.EX_PRICE: 0,
                R3.EX_TIME: 0,
                R3.EX_TYPE: 0,
            },
            R3.START_TIME: datetime.now(),
            Params.ENDTIME: 0,
            R3.UPD_INFO: {},
        }

        positions_collec.insert_one(def_data)

        fetch_positions = positions_collec.find_one({R3._ID: _id})
        
    return fetch_positions, fetch_positions[R3._ID]


def time_checker():
    return str(datetime.now().replace(second=0, microsecond=0).time())


def fetch_data(option, strikeprice):
    # print("HII",option,strikeprice)
    option_data = script_collec.find_one(
        {
            Tick.CALENDER:  {"$in":VALID_WEEKLIES},  #WeeklyExpiry.CURRENT,
            Tick.OPTION: option,
            Tick.STRIKE: strikeprice,
        }
    )

    if option_data:

        # print(option_data)
        return option_data

    else:
        logging.error(
            f"fetch_data Error No data for {option}, {strikeprice}", exc_info=True
        )

        # error_collec.update_one(
        #     {"_id": logname},
        #     {
        #         "$set": {
        #             Err.STRATEGY: Strat.R3,
        #             Err.FUNC: "tick_data",
        #             Err.ERROR: f"No data for {option}, {strikeprice}",
        #             Err.UPD_TIME: datetime.now(),
        #         }
        #     },
        #     upsert=True,
        # )

        return None


def tick_data(token):
    data = tick_collec.find_one({R3._ID: token})

    if data:
        return data
    else:

        logging.error(f"tick_data Error No data for {token}", exc_info=True)

        # error_collec.update_one(
        #     {"_id": logname},
        #     {
        #         "$set": {
        #             Err.STRATEGY: Strat.R3,
        #             Err.FUNC: "tick_data",
        #             Err.ERROR: f"No data for {token}",
        #             Err.UPD_TIME: datetime.now(),
        #         }
        #     },
        #     upsert=True,
        # )

        return None


def Candle_Fetcher(token_dict, price):
    wait_time = datetime.now().replace(second=0, microsecond=0) + timedelta(minutes=2)

    while wait_time != datetime.now().replace(second=0, microsecond=0):
        try:
            cx = datetime.now()
            end_date = cx.replace(second=0, microsecond=0)
            start_date = end_date - timedelta(minutes=1)

            data = mastertrust.historical_data(
                token_dict=token_dict,
                start_date=start_date,
                end_date=end_date,
            )
            if data:
                logging.info(f"{token_dict=} {data=}")
                return data

        except:
            logging.error("Candle Fetcher Error", exc_info=True)

            # error_collec.update_one(
            #     {"_id": logname},
            #     {
            #         "$set": {
            #             Err.STRATEGY: Strat.R3,
            #             Err.FUNC: "Candle Fetcher",
            #             Err.ERROR: traceback.format_exc(),
            #             Err.UPD_TIME: datetime.now(),
            #         }
            #     },
            #     upsert=True,
            # )

            time.sleep(0.2)

    logging.error(
        f"Candle Fetcher Error No Data for {start_date=}, {end_date=}, Toke {token_dict=}",
        exc_info=True,
    )

    # error_collec.update_one(
    #     {"_id": logname},
    #     {
    #         "$set": {
    #             Err.STRATEGY: Strat.R3,
    #             Err.FUNC: "Candle Fetcher",
    #             Err.ERROR: f"Candle Fetcher Error No Data for {start_date=}, {end_date=}, Toke {token_dict=}",
    #             Err.UPD_TIME: datetime.now(),
    #         }
    #     },
    #     upsert=True,
    # )

    return [
        [str(start_date), price, price, price, price, 11223344],
    ]


def positions_updater(data_to_update):
    global positions, ce_buy, pe_buy, ce_buy, pe_buy,ID

    positions_collec.update_one({R3._ID: ID}, {"$set": data_to_update})

    positions = positions_collec.find_one({R3._ID: ID})

    ce_buy = positions[R3.CE_buy]
    pe_buy = positions[R3.PE_buy]

    return None


def get_traded_strikes():
    traded_ce = set()
    traded_pe = set()
    logging.info("FONDING IN HIST DB >>>>>>>>>")
    prev_trades = historical_collec.find({
        "original_id": str(params[R3._ID]),
        "date": todays_date
    })
    ce ,  pe  = {},{}
    for trade in prev_trades:
        ce = trade.get(R3.CE_buy, {})
        pe = trade.get(R3.PE_buy, {})

        if ce and ce.get(R3.SYMBOL):
            traded_ce.add(ce[R3.SYMBOL])

        if pe and pe.get(R3.SYMBOL):
            traded_pe.add(pe[R3.SYMBOL])

    return ce, pe


def get_strikes():
    # Fetch stored strikes from positions
    # ce_strikes = positions.get(R3.CE_LIST, [])
    # pe_strikes = positions.get(R3.PE_LIST, [])
    # ce_strikes = CE_LIST
    # pe_strikes = PE_LIST
    traded_ce, traded_pe = get_traded_strikes()

    while time_checker() <= params[Params.LASTENTRY]:
        try:
            cx = datetime.now()
            cdate = cx.replace(second=0, microsecond=0)
            
            # Get all weekly expiry scripts
            weekly_scripts = list(script_collec.find({Tick.CALENDER: WeeklyExpiry.CURRENT}))
            # print(len(weekly_scripts))
            CE_LIST = []
            PE_LIST = []

            for script in weekly_scripts:
                # print(script)
                token = script[Tick.TOKEN]
                ltp = tick_data(token)[Tick.LTP]
                
                if script[Tick.OPTION] == Option.CE and params[Params.LOWER_STRIKE] <= ltp <= params[Params.UPPER_STRIKE]:
                    logging.info("SCRIPTS STORED AGAIN....... ")
                    CE_LIST.append({
                        "strike": script[Tick.STRIKE],
                        "ltp": ltp
                    })

                elif script[Tick.OPTION] == Option.PE and params[Params.LOWER_STRIKE] <= ltp <= params[Params.UPPER_STRIKE]:
                    PE_LIST.append({
                        "strike": script[Tick.STRIKE],
                        "ltp": ltp
                    })
            if not CE_LIST and not PE_LIST:
                logging.warning("No CE/PE strikes within upper/lower range for today")
                # return [], []
                continue

            # Save separately in positions
            positions[R3.CE_LIST] = CE_LIST
            positions[R3.PE_LIST] = PE_LIST
            positions_updater(positions)
            logging.info(f"Stored {len(CE_LIST)} CE and {len(PE_LIST)} PE scripts for today's entry")

            cx = datetime.now()
            cdate = cx.replace(second=0, microsecond=0)
            cdate_l = cdate - timedelta(minutes=1)
            ce_strikes = CE_LIST
            pe_strikes = PE_LIST


            candle_data = nf_collec.find_one({Tick.DATE: cdate_l})

            if candle_data:
                nf_price = candle_data[CANDLE.CLOSE]
                nf_price_int = int(nf_price)
                atm = round(nf_price_int / 50) * 50
                positions[R3.NF_PRICE] = nf_price
                positions[R3.ATM] = atm
                # logging.info(f"ATM CHANGE ....{atm}")

            # Use only stored strikes instead of range
            for ce_strike in ce_strikes:
                strike = ce_strike["strike"]
                ce_data = fetch_data(Option.CE, strike)
                logging.info(f"CE_STRIKE .......{ce_strike}")
                if not ce_data:
                    continue
                if traded_ce and ce_data[Tick.TOKEN] ==  traded_ce['token']:
                    
                    logging.info(f"Skipping already traded CE strike {strike}")
                    continue

                ce_tick = tick_data(ce_data[Tick.TOKEN])
                if not ce_tick:
                    continue

                ce_ltp = ce_tick[Tick.LTP]
                logging.info(f"CE_LTP .......{ce_ltp}")
                logging.info(f"CE_STRIKE .......{ce_strike, ce_data}")

                # Already filtered when stored, optional double check
                if not (params[Params.LOWER_STRIKE] <= ce_ltp <= params[Params.UPPER_STRIKE]):
                    continue

                for pe_strike in pe_strikes:
                    logging.info(f"Comparison between {ce_strike , pe_strike}")
                    strike = pe_strike["strike"]
                    pe_data = fetch_data(Option.PE, strike)
                    if not pe_data:
                        continue
                    if traded_pe and pe_data[Tick.TOKEN] == traded_pe['token']:
                        logging.info(f"Skipping already traded PE strike {strike}")
                        continue

                    pe_tick = tick_data(pe_data[Tick.TOKEN])
                    if not pe_tick:
                        continue

                    pe_ltp = pe_tick[Tick.LTP]
                    logging.info(f"PE DATA ......{pe_data, pe_tick}")

                    diff = abs(ce_ltp - pe_ltp)
                    logging.info(f"Checking CE {ce_strike} ({ce_ltp}) PE {pe_strike} ({pe_ltp}) Diff={diff}")

                    if diff <= params[Params.DIFFERENCE]:
                        logging.info("Strike pair found")
                        return ce_tick, pe_tick, ce_ltp, pe_ltp, ce_data, pe_data
                    else:
                        logging.info(f"CE {ce_strike} LTP {ce_ltp} out of range")

                time.sleep(0.5)

        except:
            logging.error("Entry", exc_info=True)
            time.sleep(0.5)

    logging.warning("No strike found")
    return None, None, None, None, None, None


def Entry():
    # global positions

    ce_tick , pe_tick ,ce_ltp ,pe_ltp ,ce_data, pe_data = get_strikes()

   
    logging.info(f"CE_DATA ...... {ce_tick}")

    logging.info(f"PE_DATA ...... {pe_tick}")

    logging.info(f"CE_ltp ...... {ce_ltp}")

    logging.info(f"PE_ltp ...... {pe_ltp}")
    if not ce_data or not pe_data:
        logging.info("No valid strike pair found. Skipping...")
        return
        
    logging.info("--------------Entry Found -----------------------")
    odate = datetime.now()  
    # ce_tick = tick_data(ce_data[Tick.TOKEN])
    # pe_tick = tick_data(pe_data[Tick.TOKEN])

    ce_EN_PRICE = ce_ltp #+ 1 
    pe_EN_PRICE = pe_ltp #+ 1 

    ce_buy[R3.EN_ID] = str(time.perf_counter_ns())
    ce_buy[R3.EN_TIME] = odate
    ce_buy[R3.SYMBOL] = ce_data[Tick.SYMBOL]
    ce_buy[R3.ROOT] = ROOT
    ce_buy[R3.TOKEN] = ce_data[Tick.TOKEN]
    ce_buy[R3.BROKER_DATA] = ce_data[Tick.BROKER_DATA]  
    ce_buy[R3.EN_PRICE] =     ce_EN_PRICE      #ce_tick[Tick.BUY_VAL]
    ce_buy[R3.SIDE] = DbOrder.BUY
    ce_buy[R3.LOTS] = 1

    pe_buy[R3.EN_ID] = str(time.perf_counter_ns())
    pe_buy[R3.EN_TIME] = odate
    pe_buy[R3.ROOT] = ROOT
    pe_buy[R3.SYMBOL] = pe_data[Tick.SYMBOL]
    pe_buy[R3.TOKEN] = pe_data[Tick.TOKEN]
    pe_buy[R3.BROKER_DATA] = pe_data[Tick.BROKER_DATA]
    pe_buy[R3.EN_PRICE] =   pe_EN_PRICE           #pe_data[Tick.BUY_VAL]
    pe_buy[R3.SIDE] = DbOrder.BUY
    pe_buy[R3.LOTS] = 1

        
    logging.info("Buy Entry Data Fetched")
            

    ce_buy[R3.STOPLOSS_1] = ce_buy[R3.EN_PRICE]  - params[Params.STOPLOSS]  #ce_buy[R3.EN_PRICE] - 20  # Stoploss
    ce_buy[R3.TARGET] =   ce_buy[R3.EN_PRICE]  + params[Params.TARGET] #  ce_buy[R3.EN_PRICE] + 30   # Target
    ce_buy[R3.TRIGGER_1] =   (ce_buy[R3.STOPLOSS_1] + 1)

    pe_buy[R3.STOPLOSS_1] =  pe_buy[R3.EN_PRICE]  - params[Params.STOPLOSS]    #pe_buy[R3.EN_PRICE] - 20  # Stoploss
    pe_buy[R3.TARGET] =  pe_buy[R3.EN_PRICE]  + params[Params.TARGET]     #pe_buy[R3.EN_PRICE] + 30   # Target
    pe_buy[R3.TRIGGER_1] =   (pe_buy[R3.STOPLOSS_1] + 1)
    
    trades_collec.insert_many(
        [
            {
                DbOrder.CS_ID: ce_buy[R3.EN_ID],
                DbOrder.LOTS: ce_buy[R3.LOTS],
                DbOrder.COND: DbOrder.EXEC,
                DbOrder.SIDE: DbOrder.BUY,
                DbOrder.PRICE: ce_buy[R3.EN_PRICE],
                DbOrder.SYMBOL: ce_buy[R3.SYMBOL],
                DbOrder.ROOT: ROOT,
                DbOrder.TOKEN: ce_buy[R3.TOKEN],
                DbOrder.BROKER_DATA: ce_buy[R3.BROKER_DATA],
                DbOrder.ORDER_TYPE: OrderType.LIMIT,
                DbOrder.ORD_TIME: odate,
                DbOrder.UPD_TIME: 0,
                DbOrder.USERS: {},
            },
            {
                DbOrder.CS_ID: pe_buy[R3.EN_ID],
                DbOrder.LOTS: pe_buy[R3.LOTS],
                DbOrder.COND: DbOrder.EXEC,
                DbOrder.SIDE: DbOrder.BUY,
                DbOrder.PRICE: pe_buy[R3.EN_PRICE],
                DbOrder.SYMBOL: pe_buy[R3.SYMBOL],
                DbOrder.TOKEN: pe_buy[R3.TOKEN],
                DbOrder.ROOT: ROOT,
                DbOrder.BROKER_DATA: pe_buy[R3.BROKER_DATA],
                DbOrder.ORDER_TYPE: OrderType.LIMIT,
                DbOrder.ORD_TIME: odate,
                DbOrder.UPD_TIME: 0,
                DbOrder.USERS: {},
            },
        ]
    )
    ce_buy[R3.SL_ID] = str(time.perf_counter_ns() + 1)
    pe_buy[R3.SL_ID] = str(time.perf_counter_ns() + 2)

    # --- 2. Insert Stop-Loss (PLACE) Orders ---
    trades_collec.insert_many([
        {
            DbOrder.CS_ID: ce_buy[R3.SL_ID],
            DbOrder.CHECK_ID: ce_buy[R3.EN_ID],
            DbOrder.LOTS: ce_buy[R3.LOTS],
            DbOrder.COND: DbOrder.PLACE,
            DbOrder.SIDE: DbOrder.SELL,
            DbOrder.TRIGGER: ce_buy[R3.TRIGGER_1],
            DbOrder.PRICE: ce_buy[R3.STOPLOSS_1] ,
            DbOrder.SYMBOL: ce_buy[R3.SYMBOL],
            DbOrder.TOKEN: ce_buy[R3.TOKEN],
            DbOrder.ROOT: ROOT,
            DbOrder.BROKER_DATA: ce_buy[R3.BROKER_DATA],
            DbOrder.ORDER_TYPE: OrderType.SL,
            DbOrder.ORD_TIME: odate,
            # DbOrder.UPD_TIME: 0,
            # DbOrder.USERS: {},
        },
        {
            DbOrder.CS_ID: pe_buy[R3.SL_ID],
            DbOrder.CHECK_ID: pe_buy[R3.EN_ID],
            DbOrder.LOTS: pe_buy[R3.LOTS],
            DbOrder.COND: DbOrder.PLACE,
            DbOrder.SIDE: DbOrder.SELL,
            DbOrder.TRIGGER: pe_buy[R3.TRIGGER_1],
            DbOrder.PRICE: pe_buy[R3.STOPLOSS_1],
            DbOrder.SYMBOL: pe_buy[R3.SYMBOL],
            DbOrder.TOKEN: pe_buy[R3.TOKEN],
            DbOrder.ROOT: ROOT,
            DbOrder.BROKER_DATA: pe_buy[R3.BROKER_DATA],
            DbOrder.ORDER_TYPE: OrderType.SL,
            DbOrder.ORD_TIME: odate, 
        }
    ])

    positions[R3.POS] = 1
    positions[R3.UPD_INFO] = {
        R3.STATUS: "BUY ITM Entry Taken and Sl Inserted",
        R3.TIME: datetime.now(),
    }

    positions_updater(positions)
    logging.info(f"Entry Successful: CE@{ce_buy[R3.EN_PRICE]}, PE@{pe_buy[R3.EN_PRICE]}")
    # else :
    #     logging.info("THe diff is gratert tht 5 ")    
    #     continue
    # return None 

            
        # except:
        #     logging.error("Entry", exc_info=True)

            # error_collec.update_one(
            #     {"_id": logname},
            #     {
            #         "$set": {
            #             Err.STRATEGY: Strat.R3,
            #             Err.FUNC: "Entry",
            #             Err.ERROR: traceback.format_exc(),
            #             Err.UPD_TIME: datetime.now(),
            #         }
            #     },
            #     upsert=True,
            # )

    # positions[R3.UPD_INFO] = {
    #     R3.STATUS: "Last Entry Time Reached No Trades Today.",
    #     R3.TIME: datetime.now(),
    # }

    # positions_updater(positions)
    return
    # kill(getpid(), 9)


if __name__ == "__main__":

    # load_dotenv()
    TICKS_URL = #getenv("TICKS_URL")
    STRAT_URL =   #getenv("STRAT_URL")
    
    # logname = path.basename(__file__)[:-3]
    # folder = file_path_locator()
    # logging.info = log_setup(logname, folder)

    trade_client = pymongo.MongoClient(STRAT_URL)
    trades = trade_client[Strat.R3]
    positions_collec = trades[Strat.POSITIONS]
    historical_collec = trades[Strat.HISTORICAL]
    trades_collec = trades[Strat.TRADES]
    params_collec = trades[Strat.PARAMS]
    # 
    # del_prev_pos ()           
    error_db = trade_client[Strat.ERROR]
    error_collec = error_db[Strat.STRAT_ERRORS]


    params = params_collec.find_one({R3._ID: 0})
    
    arg_ID = int(argv[1]) if len(argv) > 1 else 0
    
    print(arg_ID)
    logging.basicConfig(
    filename=f'Ratio_st_{arg_ID}.log',           # Log file name
    level=logging.INFO,           # Log level: DEBUG, INFO, WARNING, ERROR, CRITICAL
    format='%(asctime)s - %(levelname)s - %(message)s'
    )
    
    if arg_ID != 0:
        arg_param = params_collec.find_one({R3._ID:arg_ID}) 
        print(arg_param)
        # params = params.copy()
        params.update(arg_param)

    else:
        params  =  params 


    params[R3._ID] = arg_ID
    data_client = pymongo.MongoClient(TICKS_URL)
    ROOT = Root.NF
    script_collec = data_client[Root.NF][Tick.SUB]
    tick_collec = data_client[Root.NF][Tick.TICKS]
    nf_collec = data_client[Tick.ONE_MIN][Root.NF]
    # 1. Get all expiries and convert to date objects
    todays_date = str(datetime.now().date())

    VALID_WEEKLIES = [WeeklyExpiry.CURRENT]
    logging.info("Using WeeklyExpiry.CURRENT from DB")

    positions, ID = fetching_positions()
    positions

    # if positions[R3.POS] == 0:
    #     ce_list = positions.get(R3.CE_LIST,[])
    #     pe_list = positions.get(R3.PE_LIST,[])


    #     if not ce_list or not pe_list:
    #         print("No strikes found in DB. Selecting daily strikes now...")
    #         # select_daily_strikes()
    #         # Refresh the local variable after updating DB
    #         positions, ID = fetching_positions()
    #     else:
    #         logging.info("Using existing strikes found in positions document.")

    # select_daily_strikes()
    ce_buy = positions[R3.CE_buy]
    pe_buy = positions[R3.PE_buy]

 

    
    pause.until(datetime.now().replace(minute=30, hour=9, microsecond=0, second=0))

    logging.info("Code Started")
    
    if (positions[R3.POS] == 0) and (time_checker() <= params[Params.LASTENTRY]):
        Entry()
        
    
    elif positions[R3.POS] == 0 and time_checker() > params[Params.LASTENTRY]:
        # archive_todays_positions()
        logging.info("No Trades Today, Last Entry Time Reached.")
        kill(getpid(), 9)
    
    elif positions[R3.POS] == 5:
        check_time = time_checker()
        if check_time < params[Params.LASTENTRY]:
            logging.info("SECOND ENTRY HAS STARTED FOR TODAY ......................")
            time.sleep(60)
            sec_position_creator()
            positions, _ = fetching_positions()
            ce_buy = positions[R3.CE_buy]
            pe_buy = positions[R3.PE_buy]

            time.sleep(20)
            Trade_count()

            # Reset POS so the loop knows we're ready for a new entry
            positions[R3.POS] = 0
            positions_updater(positions)
            Entry()

        else :
            # archive_todays_positions()
            logging.info("All Trades Squared Off for Today")
            kill(getpid(), 9)

    else:
        pass

        logging.info(" -------------------- EXIT  Started -----------------------   ")
    
    while True:
        try:
            check_time = time_checker()
            ce_tick = tick_data(ce_buy[R3.TOKEN])[Tick.LTP]
            pe_tick = tick_data(pe_buy[R3.TOKEN])[Tick.LTP]
            logging.info(f"CE_ltp : {ce_tick}")
            logging.info(f"PE_ltp : {pe_tick}")
            # POS 1: Both CE and PE are open
            if positions[R3.POS] == 1:

                ce_tick = tick_data(ce_buy[R3.TOKEN])[Tick.LTP]
                pe_tick = tick_data(pe_buy[R3.TOKEN])[Tick.LTP]
                # logging.info(f"CE_ltp : {ce_tick}")
                # logging.info(f"PE_ltp : {pe_tick}")
                # ---  1: CE Target Hit (+30) -> Close both legs ---
                if ce_tick >= ce_buy[R3.TARGET]:
                    logging.info("CE Target Hit. Squaring off both.")
                    odate = datetime.now()
                    
                    # Close CE
                    trades_collec.update_one(
                        {DbOrder.CS_ID: ce_buy[R3.SL_ID]}, 
                        {"$set": 
                         {
                            DbOrder.COND: DbOrder.CANCEL,
                            DbOrder.PRICE: ce_tick,
                            DbOrder.UPD_TIME: odate
                            }})
                    trades_collec.insert_one(
                        {
                            DbOrder.CS_ID: ce_buy[R3.EN_ID],
                            DbOrder.LOTS: ce_buy[R3.LOTS],
                            DbOrder.COND: DbOrder.EXEC,
                            DbOrder.SIDE: DbOrder.BUY,
                            DbOrder.PRICE: ce_buy[R3.EN_PRICE],
                            DbOrder.SYMBOL: ce_buy[R3.SYMBOL],
                            DbOrder.ROOT: ROOT,
                            DbOrder.TOKEN: ce_buy[R3.TOKEN],
                            DbOrder.BROKER_DATA: ce_buy[R3.BROKER_DATA],
                            DbOrder.ORDER_TYPE: OrderType.LIMIT,
                            DbOrder.ORD_TIME: odate,
                            DbOrder.UPD_TIME: 0,
                            DbOrder.USERS: {},
                        }

                    )
                    ce_buy[R3.EX_PRICE] = ce_tick
                    ce_buy[R3.EX_TIME]= odate
                    ce_buy[R3.EX_TYPE] = "TARGET"

                    # Close PE at Market
                    pe_market = tick_data(pe_buy[R3.TOKEN])[Tick.LTP]

                    trades_collec.update_one(
                        {DbOrder.CS_ID: pe_buy[R3.SL_ID]}, 
                        {"$set": {
                            DbOrder.COND: DbOrder.EXEC,
                            DbOrder.PRICE: pe_market,
                            DbOrder.UPD_TIME: odate
                            }})
                    
                    pe_buy[R3.EX_PRICE] =  pe_market
                    pe_buy[R3.EX_TIME] =   odate
                    pe_buy[R3.EX_TYPE] = "SQUARE_OFF"

                    positions[R3.POS] = 5
                    positions[R3.UPD_INFO] = {R3.STATUS: "CE Target Hit", R3.TIME: odate}
                    positions_updater(positions)
                    if check_time < params[Params.LASTENTRY]:
                        logging.info("SECOND ENTRY HAS STARTED FOR TODAY ......................")
                        time.sleep(60)
                        sec_position_creator()
                        positions, _ = fetching_positions()
                        ce_buy = positions[R3.CE_buy]
                        pe_buy = positions[R3.PE_buy]
                        time.sleep(20)
                        Trade_count()
                        Entry()
                    else:
                        logging.info("All Trades Squared Off for Today")
                        kill(getpid(), 9)

                # ---  2: PE Target Hit (+30) -> Close both legs ---
                elif pe_tick >= pe_buy[R3.TARGET]:
                    logging.info("PE Target Hit. Squaring off both.")
                    odate = datetime.now()

                    # Close PE
                    trades_collec.update_one({DbOrder.CS_ID: pe_buy[R3.SL_ID]}, 
                        {"$set": 
                        {
                            DbOrder.COND: DbOrder.CANCEL,
                            DbOrder.PRICE: pe_tick,
                            DbOrder.UPD_TIME: odate
                            }})
                    
                    trades_collec.insert_one(
                    {
                        DbOrder.CS_ID: pe_buy[R3.EN_ID],
                        DbOrder.LOTS: pe_buy[R3.LOTS],
                        DbOrder.COND: DbOrder.EXEC,
                        DbOrder.SIDE: DbOrder.BUY,
                        DbOrder.PRICE: pe_buy[R3.EN_PRICE],
                        DbOrder.SYMBOL: pe_buy[R3.SYMBOL],
                        DbOrder.TOKEN: pe_buy[R3.TOKEN],
                        DbOrder.ROOT: ROOT,
                        DbOrder.BROKER_DATA: pe_buy[R3.BROKER_DATA],
                        DbOrder.ORDER_TYPE: OrderType.LIMIT,
                        DbOrder.ORD_TIME: odate,
                        DbOrder.UPD_TIME: 0,
                        DbOrder.USERS: {},
                    }
                    )

                    pe_buy[R3.EX_PRICE] = pe_tick 
                    pe_buy[R3.EX_TIME] =  odate
                    pe_buy[R3.EX_TYPE] = "TARGET"

                    # Close CE at Market
                    ce_market = tick_data(ce_buy[R3.TOKEN])[Tick.LTP]
                    trades_collec.update_one({DbOrder.CS_ID: ce_buy[R3.SL_ID]}, 
                        {"$set": 
                        {
                            DbOrder.COND: DbOrder.EXEC,
                            DbOrder.PRICE: ce_market,
                            DbOrder.UPD_TIME: odate
                            }})
                    ce_buy[R3.EX_PRICE] = ce_market 
                    ce_buy[R3.EX_TIME] =  odate
                    ce_buy[R3.EX_TYPE] =  "SQUARE_OFF"

                    positions[R3.POS] = 5
                    positions[R3.UPD_INFO] = {R3.STATUS: "PE Target Hit", R3.TIME: odate}
                    positions_updater(positions)
                    if check_time < params[Params.LASTENTRY]:
                        logging.info("SECOND ENTRY HAS STARTED FOR TODAY ......................")
                        time.sleep(60)
                        sec_position_creator()
                        positions, _ = fetching_positions()
                        ce_buy = positions[R3.CE_buy]
                        pe_buy = positions[R3.PE_buy]
                        time.sleep(20)
                        Trade_count()
                        Entry()
                    else:
                        logging.info("All Trades Squared Off for Today")
                        kill(getpid(), 9)

                # ---  3: CE SL Hit (-20) -> Trail PE to Cost + 1 ---
                elif ce_tick <= ce_buy[R3.TRIGGER_1]:
                    logging.info("CE SL Hit. Trailing PE to Entry + 1.")
                    odate = datetime.now()
                    
                    trades_collec.update_one({DbOrder.CS_ID: ce_buy[R3.SL_ID]}, 
                        {"$set": 
                        {
                            DbOrder.COND: DbOrder.EXEC,
                            DbOrder.PRICE: ce_tick,
                            DbOrder.UPD_TIME: odate
                            }})
                    
                    ce_buy[R3.EX_PRICE] = ce_tick
                    ce_buy[R3.EX_TIME] =  odate
                    ce_buy[R3.EX_TYPE] =  DbOrder.SL

                    # Update PE Trigger to Entry + 1
                    pe_buy[R3.STOPLOSS_2] = pe_buy[R3.EN_PRICE] +1
                    pe_buy[R3.TRIGGER_2] = pe_buy[R3.STOPLOSS_2] + 0.5
                    
                    update_data = {
                        DbOrder.COND: DbOrder.MODIFY,
                        DbOrder.PRICE: pe_buy[R3.STOPLOSS_2], # Limit Price
                        DbOrder.TRIGGER: pe_buy[R3.TRIGGER_2],     # Trigger Price
                        DbOrder.UPD_TIME: odate,
                    }

                    trades_collec.update_one({DbOrder.CS_ID: pe_buy[R3.SL_ID]}, 
                    {"$set": update_data})
                    
                    positions[R3.POS] = 2 # PE remains as single 
                    positions_updater(positions)
                    positions[R3.UPD_INFO] = {R3.STATUS: "CE SL Hit - PE Trailed", R3.TIME: odate}
                    positions_updater(positions)

                # ---  4: PE SL Hit (-20) -> Trail CE to Cost + 1 ---
                elif pe_tick <= pe_buy[R3.TRIGGER_1]:
                    logging.info("PE SL Hit. Trailing CE to Entry + 1.")
                    odate = datetime.now()

                    trades_collec.update_one({DbOrder.CS_ID: pe_buy[R3.SL_ID]}, 
                        {"$set": 
                        {
                            DbOrder.COND: DbOrder.EXEC,
                            DbOrder.PRICE: pe_tick,
                            DbOrder.UPD_TIME: odate
                            }})
                    
                    pe_buy[R3.EX_PRICE] = pe_tick
                    pe_buy[R3.EX_TIME] =  odate
                    pe_buy[R3.EX_TYPE] =  DbOrder.SL

                    # Update CE Trigger to Entry + 1
                    ce_buy[R3.STOPLOSS_2] = ce_buy[R3.EN_PRICE] + 1
                    ce_buy[R3.TRIGGER_2] = ce_buy[R3.STOPLOSS_2] + 0.5
                    update_data = {
                        DbOrder.COND: DbOrder.MODIFY,
                        DbOrder.PRICE: ce_buy[R3.STOPLOSS_2], # Limit Price
                        DbOrder.TRIGGER: ce_buy[R3.TRIGGER_2],     # Trigger Price
                        DbOrder.UPD_TIME: odate,
                    }
                    trades_collec.update_one({DbOrder.CHECK_ID: pe_buy[R3.SL_ID]}, 
                    {"$set": update_data})
                    update_data = {
                        DbOrder.COND: DbOrder.MODIFY,
                        DbOrder.PRICE:  ce_buy[R3.STOPLOSS_2],
                        DbOrder.TRIGGER: ce_buy[R3.TRIGGER_2],
                        DbOrder.UPD_TIME: odate,
                    }
                    trades_collec.update_one({DbOrder.CHECK_ID: ce_buy[R3.EN_ID]}, 
                    {"$set": update_data})

                    positions[R3.POS] = 3 # Custom e for trailed CE
                    positions[R3.UPD_INFO] = {R3.STATUS: "PE SL Hit - CE Trailed", R3.TIME: odate}
                    positions_updater(positions)

            # Monitoring Trailed Leg (POS 2 = PE is active, POS 3 = CE is active)
            elif positions[R3.POS] in [2, 3]:
                active_sell = ce_buy if positions[R3.POS] == 3 else pe_buy
                # active_sell = ce_buy if positions[R3.POS] == 3 else pe_buy
                curr_ltp = tick_data(active_sell[R3.TOKEN])[Tick.LTP]

                if curr_ltp >= active_sell[R3.TARGET] or curr_ltp <= active_sell[R3.TRIGGER_2]:
                    odate = datetime.now()
                    if curr_ltp >= active_sell[R3.TARGET]:
                        reason = "TARGET"


                        trades_collec.update_one({DbOrder.CS_ID: active_sell[R3.SL_ID]}, 
                        {"$set": 
                        {DbOrder.COND: DbOrder.CANCEL,
                        DbOrder.PRICE: curr_ltp,
                        DbOrder.UPD_TIME: odate
                        }})

                        trades_collec.insert_one(
                        {
                            DbOrder.CS_ID: active_sell[R3.EN_ID],
                            DbOrder.LOTS: active_sell[R3.LOTS],
                            DbOrder.COND: DbOrder.EXEC,
                            DbOrder.SIDE: DbOrder.BUY,
                            DbOrder.PRICE: active_sell[R3.EN_PRICE],
                            DbOrder.SYMBOL: active_sell[R3.SYMBOL],
                            DbOrder.TOKEN: active_sell[R3.TOKEN],
                            DbOrder.ROOT: ROOT,
                            DbOrder.BROKER_DATA: active_sell[R3.BROKER_DATA],
                            DbOrder.ORDER_TYPE: OrderType.LIMIT,
                            DbOrder.ORD_TIME: odate,
                            DbOrder.UPD_TIME: 0,
                            DbOrder.USERS: {},
                        }
                        )
                        
                    else :
                        reason = "TRAILED_SL"
                        trades_collec.update_one({DbOrder.CS_ID: active_sell[R3.SL_ID]}, 
                            {"$set": 
                            {DbOrder.COND: DbOrder.EXEC,
                            DbOrder.PRICE: curr_ltp,
                            DbOrder.UPD_TIME: odate
                            }})
                    
                    active_sell[R3.EX_PRICE]=  curr_ltp
                    active_sell[R3.EX_TIME]=   odate
                    active_sell[R3.EX_TYPE] =  reason
                    positions[R3.POS] = 5
                    positions_updater(positions)
                    if check_time < params[Params.LASTENTRY]:
                        logging.info("SECOND ENTRY HAS STARTED FOR TODAY ......................")
                        time.sleep(60)
                        sec_position_creator()
                        positions, _ = fetching_positions()
                        ce_buy = positions[R3.CE_buy]
                        pe_buy = positions[R3.PE_buy]
                        time.sleep(20)
                        Trade_count()
                        Entry()
                    else:
                        logging.info("All Trades Squared Off for Today")
                        kill(getpid(), 9)

            # End of Day Time Lapse
            if check_time >= params[Params.ENDTIME]:
                odate = datetime.now()
                reason = "TIMELAPSE"
                if positions[R3.POS] in [1, 3]: # Close CE if open
                    ltp = tick_data(ce_buy[R3.TOKEN])[Tick.SELL_VAL]
                    ce_buy[R3.EX_TYPE] = reason
                    trades_collec.update_one({DbOrder.CS_ID: ce_buy[R3.SL_ID]},
                    {"$set": 
                    {DbOrder.COND: DbOrder.MODIFY, 
                    DbOrder.PRICE: ltp,
                    DbOrder.ORDER_TYPE:OrderType.LIMIT,
                    DbOrder.UPD_TIME: odate
                    }})
                
                if positions[R3.POS] in [1, 2]: # Close PE if open
                    ltp = tick_data(pe_buy[R3.TOKEN])[Tick.SELL_VAL]
                    pe_buy[R3.EX_TYPE] = reason
                    trades_collec.update_one({DbOrder.CS_ID: pe_buy[R3.SL_ID]}, 
                    {"$set": 
                    {
                        DbOrder.COND: DbOrder.MODIFY,
                        DbOrder.PRICE: ltp,
                        DbOrder.ORDER_TYPE:OrderType.LIMIT,
                        DbOrder.UPD_TIME: odate
                        }})
                
                positions[R3.POS] = 5
                positions_updater(positions)
                break

            # time.sleep(10)

        except:
            logging.error("Main Loop Error", exc_info=True)

            # error_collec.update_one(
            #     {"_id": logname},
            #     {
            #         "$set": {
            #             Err.STRATEGY: Strat.R3,
            #             Err.FUNC: "Main Loop",
            #             Err.ERROR: traceback.format_exc(),
            #             Err.UPD_TIME: datetime.now(),
            #         }
            #     },
            #     upsert=True,
            # )
            time.sleep(0.4)


logging.info("..................BYE BYE ITS END FOR THE DAY .................")
kill(getpid(), 9)
