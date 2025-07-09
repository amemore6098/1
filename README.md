# 1import numpy as np
import sounddevice as sd
import curses
from scipy.fft import rfft, rfftfreq

# Audio settings
SAMPLE_RATE = 44100
BLOCKSIZE = 1024
CHANNELS = 1
DURATION = 0.1  # Seconds per frame

# ASCII characters for visualization (from darkest to lightest)
ASCII_GRADIENT = " .:-=+*#%@"

def audio_callback(indata, frames, time, status, screen):
    """Callback function for audio stream"""
    if status:
        print(f"Status: {status}")
    
    if any(indata):
        # Convert audio data to mono and normalize
        audio_data = indata[:, 0]
        audio_data = audio_data / np.max(np.abs(audio_data))
        
        # Generate waveform visualization
        visualize_waveform(audio_data, screen)
        
        # Compute FFT and generate spectrum visualization
        fft_data = rfft(audio_data)
        fft_freq = rfftfreq(len(audio_data), 1 / SAMPLE_RATE)
        visualize_spectrum(fft_data, fft_freq, screen)
        
        # Add peak meter
        visualize_peak_meter(audio_data, screen)
        
        screen.refresh()
    else:
        screen.addstr(0, 0, "No audio input")
        screen.refresh()

def visualize_waveform(audio_data, screen):
    """Visualize audio waveform"""
    max_y, max_x = screen.getmaxyx()
    center_y = max_y // 3
    
    # Draw waveform title
    screen.addstr(0, 2, "WAVEFORM", curses.A_BOLD)
    
    # Draw waveform
    for i in range(min(len(audio_data), max_x - 4)):
        value = audio_data[i]
        y_pos = int(center_y - value * (center_y - 2))
        try:
            # Select ASCII character based on amplitude
            char_idx = int(abs(value) * (len(ASCII_GRADIENT) - 1))
            char = ASCII_GRADIENT[char_idx]
            screen.addch(y_pos, i + 2, char)
        except:
            pass  # Ignore out-of-bounds errors

def visualize_spectrum(fft_data, fft_freq, screen):
    """Visualize audio spectrum"""
    max_y, max_x = screen.getmaxyx()
    start_y = max_y // 3 + 2
    height = max_y // 3 - 2
    
    # Draw spectrum title
    screen.addstr(start_y - 2, 2, "FREQUENCY SPECTRUM", curses.A_BOLD)
    
    # Calculate frequency bins
    bins = min(max_x - 4, 100)
    freq_bins = np.linspace(0, SAMPLE_RATE // 2, bins + 1)
    spectrum = np.zeros(bins)
    
    # Map FFT data to frequency bins
    for i in range(len(fft_freq) - 1):
        if fft_freq[i] > SAMPLE_RATE // 2:
            break
        bin_idx = int(fft_freq[i] * bins / (SAMPLE_RATE // 2))
        if bin_idx < bins:
            spectrum[bin_idx] += np.abs(fft_data[i])
    
    # Normalize spectrum
    if np.max(spectrum) > 0:
        spectrum = spectrum / np.max(spectrum)
    
    # Draw spectrum bars
    for i in range(bins):
        value = spectrum[i]
        bar_height = int(value * height)
        for j in range(bar_height):
            try:
                # Select ASCII character based on intensity
                char_idx = int(j * (len(ASCII_GRADIENT) - 1) / height)
                char = ASCII_GRADIENT[char_idx]
                screen.addch(start_y + height - j - 1, i + 2, char)
            except:
                pass  # Ignore out-of-bounds errors

def visualize_peak_meter(audio_data, screen):
    """Visualize peak meter"""
    max_y, max_x = screen.getmaxyx()
    start_y = max_y * 2 // 3 + 2
    
    # Draw peak meter title
    screen.addstr(start_y - 2, 2, "PEAK METER", curses.A_BOLD)
    
    # Calculate peak and RMS
    peak = np.max(np.abs(audio_data))
    rms = np.sqrt(np.mean(audio_data**2))
    
    # Draw peak bar
    peak_width = int(peak * (max_x - 10))
    rms_width = int(rms * (max_x - 10))
    
    # Clear previous bar
    screen.addstr(start_y, 2, " " * (max_x - 4))
    
    # Draw RMS bar (green)
    screen.addstr(start_y, 2, "=" * rms_width, curses.color_pair(2))
    
    # Draw peak bar (red if clipping)
    color_pair = 3 if peak >= 0.9 else 1
    screen.addstr(start_y, 2 + rms_width, "=" * (peak_width - rms_width), curses.color_pair(color_pair))
    
    # Draw text indicators
    screen.addstr(start_y, max_x - 10, f"PEAK: {peak:.2f}")

def main(stdscr):
    # Initialize screen
    stdscr.clear()
    stdscr.nodelay(1)  # Non-blocking input
    curses.curs_set(0)  # Hide cursor
    
    # Initialize colors
    curses.start_color()
    curses.init_pair(1, curses.COLOR_WHITE, curses.COLOR_BLACK)  # Normal
    curses.init_pair(2, curses.COLOR_GREEN, curses.COLOR_BLACK)  # RMS
    curses.init_pair(3, curses.COLOR_RED, curses.COLOR_BLACK)    # Peak/clipping
    
    # Draw instructions
    stdscr.addstr(curses.LINES - 2, 2, "Press 'q' to quit", curses.A_BOLD)
    
    # Start audio stream
    try:
        with sd.InputStream(
            samplerate=SAMPLE_RATE,
            blocksize=BLOCKSIZE,
            channels=CHANNELS,
            callback=lambda *args: audio_callback(*args, stdscr)
        ):
            while True:
                # Check for user input
                key = stdscr.getch()
                if key == ord('q'):
                    break
                
                # Refresh screen
                stdscr.refresh()
                time.sleep(0.01)
    
    except Exception as e:
        stdscr.addstr(0, 0, f"Error: {str(e)}")
        stdscr.refresh()
        stdscr.getch()

if __name__ == "__main__":
    curses.wrapper(main)
