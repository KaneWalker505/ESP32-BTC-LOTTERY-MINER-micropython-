import network
import socket
import json
import hashlib
import time
import binascii

# --- CONFIGURATION ---
WIFI_SSID = "Your_WiFi_Name"
WIFI_PASS = "Your_WiFi_Password"
BITCOIN_ADDRESS = "your_bitcoin_address_here"
POOL_ADDRESS = "solo.ckpool.org"
POOL_PORT = 3333

def connect_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if not wlan.isconnected():
        print("Connecting to WiFi...")
        wlan.connect(WIFI_SSID, WIFI_PASS)
        while not wlan.isconnected():
            pass
    print("WiFi Connected! IP:", wlan.ifconfig())

def reverse_bytes(hex_str):
    """Swaps endianness of a hex string to match Bitcoin block header formatting."""
    if not isinstance(hex_str, str):
        hex_str = str(hex_str)
    if len(hex_str) % 2 != 0:
        hex_str = "0" + hex_str
    
    b = binascii.unhexlify(hex_str)
    # FIX: Avoid [::-1] step slice. Reversing bytes via a list layout instead.
    reversed_b = bytearray(len(b))
    for i in range(len(b)):
        reversed_b[i] = b[len(b) - 1 - i]
        
    return binascii.hexlify(reversed_b).decode()

def read_clean_line(sock):
    """Safely accumulates streaming socket bytes until a true newline is found."""
    buffer = bytearray()
    while True:
        data = sock.recv(1)
        if not data:
            return None
        buffer.extend(data)
        if data == b'\n':
            try:
                return buffer.decode('utf-8').strip()
            except:
                return None

def mine():
    connect_wifi()
    
    s = socket.socket()
    addr = socket.getaddrinfo(POOL_ADDRESS, POOL_PORT)[-1][-1]
    s.connect(addr)
    print("Connected to Pool")

    # 1. Subscribe to get Extranonce
    s.send(b'{"id": 1, "method": "mining.subscribe", "params": []}\n')
    
    extranonce1 = None
    extranonce2_size = 4  
    
    while True:
        line = read_clean_line(s)
        if not line:
            print("Failed to get response from pool.")
            return
            
        try:
            response = json.loads(line)
            if response.get("id") == 1:
                result = response.get("result")
                if result and len(result) >= 3:
                    extranonce1 = str(result[1])       
                    extranonce2_size = int(result[2])  
                    print(f"Subscribed! Extranonce1: {extranonce1}, Size: {extranonce2_size}")
                    break
            elif response.get("method") == "mining.set_difficulty":
                print(f"Received initial difficulty: {response['params']}")
            elif response.get("method") == "mining.set_extranonce":
                extranonce1 = str(response["params"][0])
                extranonce2_size = int(response["params"][1])
                print(f"Set Extranonce Override: {extranonce1}")
        except Exception as e:
            pass

    if not extranonce1:
        print("Could not retrieve Extranonce1. Aborting.")
        return

    extranonce2 = "00" * extranonce2_size 

    # 2. Authorize Miner
    auth_msg = '{"id": 2, "method": "mining.authorize", "params": ["' + BITCOIN_ADDRESS + '", "x"]}\n'
    s.send(auth_msg.encode())
    
    while True:
        line = read_clean_line(s)
        if not line: break
        try:
            res = json.loads(line)
            if res.get("id") == 2:
                print("Authorized wallet successfully. Mining core active...")
                break
        except:
            pass

    # 3. Main Work Loop
    while True:
        line = read_clean_line(s)
        if not line:
            break
        
        try:
            data = json.loads(line)
            if data.get("method") == "mining.notify":
                params = data["params"]
                
                job_id = str(params[0])
                prevhash = str(params[1])
                coinb1 = str(params[2])
                coinb2 = str(params[3])
                merkle_branch = params[4]
                
                version = f"{params[5]}" if isinstance(params[5], str) else f"{params[5]:08x}"
                nbits = f"{params[6]}" if isinstance(params[6], str) else f"{params[6]:08x}"
                ntime = f"{params[7]}" if isinstance(params[7], str) else f"{params[7]:08x}"
                
                print(f"\n--- New Job: {job_id} ---")
                
                # --- MINING CORE: RECONSTRUCT BLOCK HEADER ---
                coinbase_hex = coinb1 + extranonce1 + extranonce2 + coinb2
                coinbase_bin = binascii.unhexlify(coinbase_hex)
                coinbase_hash = hashlib.sha256(hashlib.sha256(coinbase_bin).digest()).digest()
                
                merkle_root = coinbase_hash
                for branch in merkle_branch:
                    branch_bin = binascii.unhexlify(str(branch))
                    merkle_root = hashlib.sha256(hashlib.sha256(merkle_root + branch_bin).digest()).digest()
                merkle_root_hex = binascii.hexlify(merkle_root).decode()
                
                header_base = (
                    reverse_bytes(version) +
                    reverse_bytes(prevhash) +
                    reverse_bytes(merkle_root_hex) +
                    reverse_bytes(ntime) +
                    reverse_bytes(nbits)
                )
                header_base_bin = binascii.unhexlify(header_base)
                
                target = int(nbits[2:], 16) * (2 ** (8 * (int(nbits[:2], 16) - 3)))
                
                # --- HASHING LOOP ---
                print(f"Hashing job:{job_id}")
                start_time = time.time()
                
                for nonce in range(100000):
                    nonce_bin = nonce.to_bytes(4, 'little')
                    block_header = header_base_bin + nonce_bin
                    
                    hash1 = hashlib.sha256(block_header).digest()
                    hash2 = hashlib.sha256(hash1).digest()
                    
                    # FIX: Avoid hash2[::-1] by manually building big-endian integer comparison byte-by-byte
                    # (Standard Bitcoin blocks are evaluated with Little-Endian hashing targets)
                    reversed_hash = bytearray(32)
                    for i in range(32):
                        reversed_hash[i] = hash2[31 - i]
                    
                    hash_int = int.from_bytes(reversed_hash, 'big')
                    
                    if hash_int < target:
                        print(f"!!! HASH FOUND !!! At nonce: {nonce}")
                        
                        # Reversing nonce bytes safely for pool submit formatting
                        rev_nonce_bin = bytearray(4)
                        for i in range(4):
                            rev_nonce_bin[i] = nonce_bin[3 - i]
                        nonce_hex = binascii.hexlify(rev_nonce_bin).decode()
                        
                        submit_msg = {
                            "id": 4,
                            "method": "mining.submit",
                            "params": [BITCOIN_ADDRESS, job_id, extranonce2, ntime, nonce_hex]
                        }
                        s.send((json.dumps(submit_msg) + "\n").encode())
                        print("Block solution submitted to pool.")
                        break
                        
                duration = time.time() - start_time
                print(f"Loop finished. Speed: {100000 / duration:.2f} H/s")
                
        except Exception as e:
            print("Mining loop error:", e)

if __name__ == "__main__":
    mine()
