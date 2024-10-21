# python-ssl-check-auto

1. make sure you was installed python3
2. Create files ```domains.txt```
   ```
3. on domains.txt input domain what you want make separate with new line, example :
google.com
facebook.com
   ```
4. make file ``` run.py ```Copy paste this code
```python
import subprocess
import re
import time
from concurrent.futures import ThreadPoolExecutor

# Read the list of domains from a file
with open('domains.txt', 'r') as file:
    domains = [line.strip() for line in file if line.strip()]

# Function to run sslscan and retrieve certificate details
def check_ssl(domain):
    print(f"Checking SSL for: {domain}")
    
    try:
        # Run the sslscan command
        result = subprocess.run(["sslscan", "--no-colour", domain], stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
        output = result.stdout
        
        # Extract Not valid before and Not valid after information
        not_valid_before = re.search(r'Not valid before: (.*?)\n', output)
        not_valid_after = re.search(r'Not valid after: (.*?)\n', output)
        
        # Calculate remaining days
        if not_valid_after:
            expiry_date = time.strptime(not_valid_after.group(1).strip(), "%b %d %H:%M:%S %Y GMT")
            expiry_timestamp = time.mktime(expiry_date)
            days_left = (expiry_timestamp - time.time()) / (60 * 60 * 24)
            days_left = round(days_left)
        else:
            days_left = 'Not found'
        
        return {
            'domain': domain,
            'not_valid_before': not_valid_before.group(1).strip() if not_valid_before else 'Not found',
            'not_valid_after': not_valid_after.group(1).strip() if not_valid_after else 'Not found',
            'days_left': days_left
        }
    
    except Exception as e:
        print(f"Error checking {domain}: {e}")
        return {
            'domain': domain,
            'error': str(e)
        }

# Save results to a file
def save_results(results):
    with open('ssl_results.txt', 'a') as result_file:
        for result in results:
            if isinstance(result['days_left'], int):
                if result['days_left'] > 365:
                    years_left = result['days_left'] // 365
                    months_left = (result['days_left'] % 365) // 30
                    days_left_str = f"{years_left} years, {months_left} months"
                else:
                    days_left_str = f"{result['days_left']} days"
            else:
                days_left_str = result['days_left']
            
            line = f"{result['domain']}\t{result['not_valid_before']}\t{result['not_valid_after']}\t{days_left_str}\n"
            result_file.write(line)

# Function to handle multi-threading SSL checks
def run_ssl_checks(domains):
    with ThreadPoolExecutor(max_workers=5) as executor:
        results = list(executor.map(check_ssl, domains))
    return results

# Run SSL checks for each domain using multi-threading
if __name__ == '__main__':
    results = run_ssl_checks(domains)
    save_results(results)
    print("SSL check completed. Results saved in ssl_results.txt.")

```
5. Run ```python3 run.py```
