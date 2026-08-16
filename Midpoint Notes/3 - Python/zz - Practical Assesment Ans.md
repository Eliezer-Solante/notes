```python
"""
solutions.py — SRE on-call helper functions
"""


# ---------------------------------------------------------------------------
# Task 1 – Log Level Summary
# ---------------------------------------------------------------------------
def summarize_log_levels(file_path: str):
    error_logs = {}
    with open(file_path, 'r') as file:
        for error in file:
            if not error.strip():
                continue
            splitted = error.split()
            for word in splitted:
                if word.isupper():
                    error_logs[word] = error_logs.get(word, 0) + 1
                    break
    return error_logs


# ---------------------------------------------------------------------------
# Task 2 – Password Strength Evaluator
# ---------------------------------------------------------------------------
def evaluate_password(password: str):
    if len(password) < 8:
        return "WEAK"

    has_upper = False
    has_lower = False
    has_digit = False

    for char in password:
        if char.isupper():
            has_upper = True
        if char.islower():
            has_lower = True
        if char.isdigit():
            has_digit = True

    if not has_upper or not has_lower or not has_digit:
        return "WEAK"

    return "STRONG"


# ---------------------------------------------------------------------------
# Task 3 – Service Health Checker
# ---------------------------------------------------------------------------
def check_service_health(response_times: list[int]):
    if not response_times:
        return "HEALTHY"

    for i in range(len(response_times)):
        average = sum(response_times[:i + 1]) / (i + 1)
        if average > 300:
            return "UNHEALTHY"

    return "HEALTHY"


# ---------------------------------------------------------------------------
# Task 4 – Detect Flapping Services
# ---------------------------------------------------------------------------
def is_service_flapping(statuses: list[str]):
    streak = 0

    for i in range(1, len(statuses)):
        if statuses[i] != statuses[i - 1]:
            streak += 1
            if streak >= 3:
                return True
        else:
            streak = 0

    return False


# ---------------------------------------------------------------------------
# Task 5 – Extract ERROR Messages
# ---------------------------------------------------------------------------
def get_error_logs(logs: list[str]):
    return [log for log in logs if log.startswith('ERROR')]


# ---------------------------------------------------------------------------
# Task 6 – Service Latency Analyzer
# ---------------------------------------------------------------------------
def analyze_latencies(services: list):
    latency_a = {}

    total = 0
    count = 0
    biggest = 0
    list_of_biggest = []
    good_count = 0

    for service in services:
        lat = service['latency']
        total = total + lat
        count = count + 1

        if lat > biggest:
            biggest = lat

        if lat <= 300:
            good_count = good_count + 1

    avg = total / count
    avg = round(avg, 2)

    for service in services:
        if service['latency'] == biggest:
            list_of_biggest.append(service['name'])

    latency_a['average'] = avg
    latency_a['top_latency_services'] = list_of_biggest
    latency_a['within_limits'] = good_count

    return latency_a


if __name__ == "__main__":
    print(summarize_log_levels('summarize_log_levels.log'))
    print(evaluate_password('Abc123!@'))
    print(check_service_health([120, 200, 650, 180, 700]))
    print(is_service_flapping(['UP', 'DOWN', 'UP', 'DOWN', 'UP']))
    print(get_error_logs(['INFO Start', 'ERROR Database connection failed', 'ERROR Timeout occurred']))
    print(analyze_latencies([{'name': 'AuthService', 'latency': 250}, {'name': 'PaymentService', 'latency': 350}, {'name': 'OrderService', 'latency': 350}]))

```
