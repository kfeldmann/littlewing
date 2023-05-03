#!/usr/bin/env python

"""
Littlewing: Simple, powerful infrastructure automation.
https://github.com/kfeldmann/littlewing

Copyright (c) 2016, Kris Feldmann
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions
are met:

  1. Redistributions of source code must retain the above copyright
     notice, this list of conditions and the following disclaimer.

  2. Redistributions in binary form must reproduce the above
     copyright notice, this list of conditions and the following
     disclaimer in the documentation and/or other materials provided
     with the distribution.

  3. Neither the name of the copyright holder nor the names of its
     contributors may be used to endorse or promote products derived
     from this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
"AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED
TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A
PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER
OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF
LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
"""

import sys
import json
import os
import re
import subprocess
import time

import yaml

if sys.version_info > (3, 0):
    from urllib.parse import quote, unquote
else:
    from urllib import quote, unquote

AWSCLI = os.environ.get('AWSCLI', 'aws')
FILENAME = '(none)'
LINENO = '(none)'
STEPNO = '(none)'
TS = time.strftime('%Y%m%d%H%M%S', time.localtime())

REGION = None
PROFILE = None
VAR = {}
SECRETS = {}
TARGET_GROUPS = {}
MARKER = 'tilde'
LOOP_STACK = []
NULL_LOOP_DEPTH = 0

LISTING = False
TARGET = None
DEFAULT_FOUND = False
OUTDIR = '.lw-output'

def _err(message):
    """
    Write error message to stderr along with FILENAME,
    STEPNO, and LINENO, and then exit(1).
    """
    sys.stderr.write('ERROR: file: %s, step: %s, line: %s\n' \
                    % (FILENAME, STEPNO, LINENO))
    sys.stderr.write('ERROR: %s\n' % message)
    raise SystemExit(1)

def _wrn(message):
    """
    Write warning message to stderr along with FILENAME,
    STEPNO, and LINENO.
    """
    sys.stderr.write('WARN: file: %s, step: %s, line: %s\n' \
                    % (FILENAME, STEPNO, LINENO))
    sys.stderr.write('WARN: %s\n' % message)
    return None

def _inf(message):
    "Write informational message to stdout."
    sys.stdout.write('INFO: %s\n' % message)
    return None

def _dbg(message):
    """
    If the environment variable DEBUG has been set, write
    debugging message to stdout.
    """
    if os.environ.get('DEBUG', False):
        sys.stdout.write('DEBUG: %s\n' % message)
    return None

def usage():
    "Write usage statement to stderr and exit(2)."
    sys.stderr.write('Usage: lw [-l|--list|<target>]\n')
    raise SystemExit(2)

def __var_asm(sec_name):
    sec_response = aws_call( \
      REGION,
      PROFILE,
      ('secretsmanager', 'get-secret-value'),
      ('--secret-id', sec_name) \
    )
    ss = sec_response.get('SecretString', '')
    if ss != '':
        SECRETS[ss] = 1
    return ss

def __varsub(in_str):
    "Dereference variables and output references."
    global FILENAME, LINENO
    _dbg('__varsub(): starting with in_str = %s' % in_str)
    words = in_str.split('.')
    _dbg('__varsub(): words = %s' % repr(words))
    if words[0] == 'var':
        if len(words) == 2:
            if words[1] in VAR:
                return VAR[words[1]]
            raise ValueError('Unknown variable: %s' % words[1])
        if len(words) == 3:
            if words[1] in VAR:
                if isinstance(VAR[words[1]], dict) and words[2] in VAR[words[1]]:
                    return VAR[words[1]][words[2]]
                if isinstance(VAR[words[1]], str):
                    raise ValueError('Treating a string variable as a map: %s.%s'
                                     % (words[1], words[2]))
                raise ValueError('Unknown variable: %s.%s'
                                 % (words[1], words[2]))
            raise ValueError('Unknown variable: %s.%s'
                             % (words[1], words[2]))
        raise ValueError('Malformed variable line: %s' % in_str)
    if words[0] == 'aws':
        fname = '.'.join(words[:4])
    elif words[0] == 'exec':
        fname = '.'.join(words[:3])
    elif words[0] == 'asm':
        return __var_asm('.'.join(words[1:]))
    else:
        _err('Unknown provider: %s' % words[0])
    try:
        f = open('%s/%s' % (OUTDIR, fname), 'r')
        FILENAME = '%s/%s' % (OUTDIR, fname)
    except IOError:
        _err('Unknown resource: %s' % fname)
    LINENO = 0
    while True:
        line = f.readline()
        LINENO += 1
        if line == '':
            break
        line = line.strip()
        try:
            n, v = line.split(':', 1)
        except ValueError:
            f.close()
            _err('Malformed line in output file (%s): %s'
                 % (fname, line))
        if n == in_str:
            f.close()
            return unquote(v)
    f.close()
    _err('Unknown resource attribute: %s' % in_str)

def varsub(val):
    """
    Recursively (to a limit) find variables in the input string
    and call __varsub() to dereference them.
    """
    _dbg('varsub(): starting with val = %s' % val)
    if isinstance(val, None.__class__):
        return val
    if isinstance(val, int):
        return str(val)
    i = 0
    while i < 100:
        if MARKER == 'dollar':
            s = re.search(r'\$\{([^${}]+)\}', val)
        else:
            s = re.search(r'\~\{([^~{}]+)\}', val)
        try:
            s.group(1)
        except (IndexError, AttributeError):
            break
        _dbg('varsub(): s.group(0) = %s' % s.group(0))
        _dbg('varsub(): s.group(1) = %s' % s.group(1))
        needle = s.group(0).replace('$', r'\$')
        val = re.sub(needle, __varsub(s.group(1)), val)
        i += 1
    if MARKER == 'dollar':
        # Un-escape $\\{ --> ${, and $\\\{ --> $\{
        val = val.replace(r'$\\{', r'${')
        val = val.replace(r'$\\\{', r'$\{')
    return val

def aws_call(region, profile, cmdtuple, argtuple):
    "Spawn aws-cli subprocess to execute commands."
    if (not region) or (not profile):
        _err('Must set region and profile before calling aws.')
    args = [
        AWSCLI,
        '--region', region,
        '--profile', profile
    ]
    args.extend(cmdtuple)
    if argtuple:
        args.extend(argtuple)
    try:
        outdata = subprocess.check_output(args)
    except (OSError, subprocess.CalledProcessError) as ex:
        _err('Failed to run %s %s %s: %s\n' \
             % (AWSCLI, cmdtuple[0], cmdtuple[1], ex) \
             + 'ERROR: Args: %s' % repr(args))
    if outdata: # Some calls do not return data
        if cmdtuple[0] == 's3':
            # Output is text, not JSON
            return {'Output': outdata}
        try:
            outdict = json.loads(outdata)
        except json.JSONDecodeError as ex:
            _err( \
            """Failed to parse json response from %s %s %s.
            Exception is: %s
            JSON source:
            %s
            """ % (AWSCLI, cmdtuple[0], cmdtuple[1], ex, outdata))
        outdict['Output'] = outdata
    else:
        outdict = None
    return outdict

def exec_call(cmd, argtuple):
    "Spawn subprocess to execute <cmd>."
    args = [cmd]
    if argtuple:
        args.extend(argtuple)
    try:
        outdata = subprocess.check_output(args)
    except (OSError, subprocess.CalledProcessError) as ex:
        _err('Failed to run %s: %s' % (cmd, ex))
    return {'Output': outdata}

def redact(s):
    """
    Look for secret values in s and redact them.
    """
    for sec in SECRETS.keys():
        s = s.replace(sec, 'redacted')
    return s

def stringify(path, name, obj):
    """
    Recursively convert heirarchical object into a list of strings,
    each using a dotted notation to represent the branch to each
    attribute and its value.
    """
    strings = []
    if obj is None:
        return strings
    if isinstance(obj, dict):
        for k in obj:
            strings.extend(stringify('%s.%s' % (path, name), k, obj[k]))
    elif isinstance(obj, list):
        i = 0
        for item in obj:
            strings.extend(stringify('%s.%s' % (path, name), i, item))
            i += 1
        this = '%s.%s.Count:%d' % (path, name, len(obj))
        strings.append(this.lstrip('.'))
    elif isinstance(obj, bytes):
        this = '%s.%s:%s' % (path, name, quote(redact(obj.decode('utf-8'))))
        strings.append(this.lstrip('.'))
    else:
        this = '%s.%s:%s' % (path, name, quote(redact(str(obj))))
        strings.append(this.lstrip('.'))
    return strings

def store_output(name, outdict):
    "Stringify and write out the output data from a step."
    try:
        f = open('%s/%s' % (OUTDIR, name), 'w')
    except IOError:
        _err('Failed to open output file: %s' % name)
    for line in stringify('', name, outdict):
        f.write('%s\n' % line)
    f.close()
    return None

def clear_all_output():
    "Move all output aside, effectively clearing it."
    global TS
    if not os.path.exists('%s-cleared-%s' % (OUTDIR, TS)):
        os.mkdir('%s-cleared-%s' % (OUTDIR, TS))
    for f in os.listdir(OUTDIR):
        os.rename( \
          '%s/%s' % (OUTDIR, f),
          '%s-cleared-%s/%s' % (OUTDIR, TS, f))
    return None

def clear_one_output(path):
    "Move one output file aside, effectively clearing it."
    global TS
    if not os.path.exists('%s/%s' \
                      % (OUTDIR, path)):
        _err("Output not found: %s. Was it already cleared?" % path)
    else:
        if not os.path.exists('%s-cleared-%s' % (OUTDIR, TS)):
            os.mkdir('%s-cleared-%s' % (OUTDIR, TS))
        os.rename( \
          '%s/%s' % (OUTDIR, path),
          '%s-cleared-%s/%s' % (OUTDIR, TS, path))
    return None

def varsub_tuple(intuple):
    "Substitute variables in each element of a tuple."
    if intuple is None:
        return None
    outtuple = []
    for item in intuple:
        try:
            outtuple.append(varsub(item))
        except ValueError as ex:
            _err(str(ex))
    return outtuple

def set_region(value, tries=1, trylimit=1):
    """
    Set the global 'REGION` and VAR['REGION'] variables
    using variable substitution.
    """
    global REGION
    try:
        REGION = varsub(value)
        VAR['REGION'] = REGION
    except ValueError as ex:
        if tries >= trylimit:
            _err(str(ex))
        else:
            return False
    return True

def set_profile(value, tries=1, trylimit=1):
    """
    Set the global 'PROFILE` and VAR['PROFILE'] variables
    using variable substitution.
    """
    global PROFILE
    try:
        PROFILE = varsub(value)
        VAR['PROFILE'] = PROFILE
    except ValueError as ex:
        if tries >= trylimit:
            _err(str(ex))
        else:
            return False
    return True

def start_loop(this_stepno, value, tries=1, trylimit=1):
    """
    Push the start and count of this loop onto the LOOP_STACK.
    Use variable substitution for the length.
    The values in LOOP_STACK[] have the following format:
    [starting_stepno, loop_count, loop_index]
    The loop_index value is incremented during the loop
    execution until it is equal to loop_count.
    """
    try:
        count = varsub(value)
    except ValueError as ex:
        if tries >= trylimit:
            _err(str(ex))
        else:
            return False
    else:
        try:
            count = int(count)
        except ValueError as ex:
            _err(str(ex))
        else:
            _dbg('Pushing loop onto the stack: [%d, %d, 0]' \
                % (this_stepno, count))
            LOOP_STACK.append([this_stepno, count, 0])
    set_loop_index_var()
    return True

def check_null_loop():
    """
    Are we inside a loop that is already at it's limit? For example,
    a zero-length loop which can be used as a poor-man's boolean.
    """
    if LOOP_STACK:
        if LOOP_STACK[-1][2] >= LOOP_STACK[-1][1]:
            _dbg('check_null_loop() returning True')
            return True
    else:
        _dbg('check_null_loop() returning False')
        return False
    return None # only for pylint

def add_null_loop():
    """
    Add a "null loop" to the stack (actually just a counter).
    """
    global NULL_LOOP_DEPTH
    NULL_LOOP_DEPTH += 1
    _dbg('Added a null loop')
    return None

def end_null_loop():
    """
    Remove a "null loop" from the stack (actually just a counter).
    """
    global NULL_LOOP_DEPTH
    if NULL_LOOP_DEPTH <= 0:
        return False
    NULL_LOOP_DEPTH -= 1
    _dbg('Subtracted a null loop')
    return True

def increment_loop_index():
    """
    Increment the loop_index value.
    Return True if the loop should continue, and False if it has reached
    the end (index == count).
    """
    LOOP_STACK[-1][2] += 1
    _dbg('Loop status: %d, %d, %d' \
        % (LOOP_STACK[-1][0], LOOP_STACK[-1][1], LOOP_STACK[-1][2]))
    if LOOP_STACK[-1][2] >= LOOP_STACK[-1][1]:
        _dbg('Loop done. Popping from LOOP_STACK.')
        LOOP_STACK.pop()
        set_loop_index_var()
        return False
    _dbg('Loop continuing.')
    set_loop_index_var()
    return True

def set_loop_index_var():
    """
    Update the LOOP_INDEX* variables, which indicate how many times
    each loop has iterated so far.
    """
    if LOOP_STACK:
        VAR['LOOP_INDEX'] = str(LOOP_STACK[-1][2])
        _dbg('Setting LOOP_INDEX to %s' % str(LOOP_STACK[-1][2]))
        if len(LOOP_STACK) > 1:
            j = 1
            for i in range(len(LOOP_STACK)-2, -1, -1):
                VAR['LOOP_INDEX_%d' % j] = str(LOOP_STACK[i][2])
                _dbg('i=%d, Setting LOOP_INDEX_%d to %s' \
                     % (i, j, str(LOOP_STACK[i][2])))
                j += 1
    else:
        VAR['LOOP_INDEX'] = "0"
    return None

def set_var(name, value, tries=1, trylimit=1):
    "Set global 'VAR[name]' using variable substitution."
    if isinstance(value, str):
        try:
            VAR[name] = varsub(value)
            _dbg('Set string VAR[%s] = "%s"' % (name, VAR[name]))
        except ValueError as ex:
            if tries >= trylimit:
                _err(str(ex))
            else:
                return False
    elif isinstance(value, int):
        VAR[name] = str(value)
        _dbg('Set int VAR[%s] = "%s"' % (name, VAR[name]))
    elif isinstance(value, dict):
        VAR[name] = value
        _dbg('Set dict VAR[%s] = "%s"' % (name, repr(VAR[name])))
    else:
        _err('Unknown variable type for var.%s' % name)
    return True

def set_default_target(value, tries=1, trylimit=1):
    """
    Set global 'TARGET' (once) using variable substitution.
    """
    global TARGET
    global DEFAULT_FOUND

    if not TARGET:
        if isinstance(value, str):
            try:
                TARGET = varsub(value)
                if LISTING:
                    DEFAULT_FOUND = True
            except ValueError as ex:
                if tries >= trylimit:
                    _err(str(ex))
                else:
                    return False
        else:
            _wrn('Default target must be a string.')
        return True
    return None # only for pylint

def set_target_groups(tgdict, tries=1, trylimit=1):
    "Set global 'TARGET_GROUPS' using variable substitution."
    if not isinstance(tgdict, dict):
        _wrn("'target-groups' must be a dict.")
        return True
    for tgname in tgdict:
        if not isinstance(tgdict[tgname], list):
            _wrn("'target-groups: %s' must be a list." % tgname)
            return True
        try:
            tgn = varsub(tgname)
        except ValueError as ex:
            if tries >= trylimit:
                _err(str(ex))
            else:
                return False
        else:
            try:
                TARGET_GROUPS[tgn] = varsub_tuple(tgdict[tgname])
            except ValueError as ex:
                if tries >= trylimit:
                    _err(str(ex))
                else:
                    return False
    return True

### main ###############################################################

if len(sys.argv) > 2 \
        or  '-?' in sys.argv \
        or '-h' in sys.argv \
        or '--help' in sys.argv:
    usage()

if len(sys.argv) == 2:
    if sys.argv[1] in ('-l', '--list'):
        LISTING = True
    else:
        TARGET = sys.argv[1]

# Legacy support
if os.path.exists('.terraflop-output'):
    OUTDIR = '.terraflop-output'

if not os.path.exists(OUTDIR):
    os.mkdir(OUTDIR)

ALL_FILES = os.listdir('.')
INPUT_FILES = []
for F in ALL_FILES:
    if F[-4:] == '.yml':
        INPUT_FILES.append(F)
INPUT_FILES.sort()

VAR['USER'] = os.environ.get('USER', 'littlewing')
VAR['LOOP_INDEX'] = "0"
VAR['LOOP_INDEX_1'] = "0"
VAR['LOOP_INDEX_2'] = "0"
VAR['LOOP_INDEX_3'] = "0"
VAR['LOOP_INDEX_4'] = "0"
VAR['LOOP_INDEX_5'] = "0"
VAR['LOOP_INDEX_6'] = "0"
VAR['LOOP_INDEX_7'] = "0"
VAR['LOOP_INDEX_8'] = "0"
VAR['LOOP_INDEX_9'] = "0"

for in_file in INPUT_FILES:
    F = open(in_file, 'r')
    FILENAME = in_file
    LINENO = '(yaml)'
    yd = yaml.load(F, Loader=yaml.Loader)
    F.close()

    # Attempt to auto-discover the variable MARKER ('$' or '~')
    F = open(in_file, 'r')
    data = F.read()
    F.close()
    dollar = data.count('${')
    tilde = data.count('~{')
    if dollar > 0 and tilde == 0:
        MARKER = 'dollar'
        _dbg('Setting variable MARKER to dollar ${}')
    elif dollar > 0 and tilde > 0:
        _wrn('Found both ${dollar} and ~{tilde} variable markers!')

    # Have to make sure we load all the vars, target-groups,
    # default-target, and set region/profile before
    # running the steps.
    if LISTING:
        _inf('File: %s' % FILENAME)

    # Try up to 'maxtries' passes to resolve variable dependencies
    z = 0
    maxtries = 10
    vardep_keep_going = False
    while True:
        z += 1
        for K in yd:
            LINENO = '(%s...)' % str(K)[:10]
            if K == 'region':
                if not set_region(yd[K], z, maxtries):
                    vardep_keep_going = True
                else:
                    _dbg('region is set to %s' % REGION)
            elif K == 'profile':
                if not set_profile(yd[K], z, maxtries):
                    vardep_keep_going = True
                else:
                    _dbg('profile is set to %s' % PROFILE)
            elif K[:4] == 'var.':
                if not set_var(K[4:], yd[K], z, maxtries):
                    vardep_keep_going = True
                else:
                    _dbg('VAR[%s] is set to %s' % (K[4:], VAR[K[4:]]))
            elif K == 'default-target':
                if not set_default_target(yd[K], z, maxtries):
                    vardep_keep_going = True
                else:
                    _dbg('default-target is set to %s' % TARGET)
            elif K == 'target-groups':
                if not set_target_groups(yd[K], z, maxtries):
                    vardep_keep_going = True
        if (z >= maxtries) or (not vardep_keep_going):
            break

    # List targets
    if LISTING:
        if DEFAULT_FOUND:
            _inf('Default target: %s' % TARGET)
        else:
            _inf('Default target: (all steps)')
        for TGNAME in TARGET_GROUPS:
            _inf('[%s]: %s' \
                % (TGNAME, ', '.join(TARGET_GROUPS[TGNAME])))
        for K in yd:
            if K[:5] == 'steps':
                try:
                    tgt = varsub(K[6:])
                except ValueError as ex:
                    _err(str(ex))
                if tgt:
                    _inf(tgt)
                else:
                    if DEFAULT_FOUND:
                        _inf('(targetless - not reachable since ' \
                            + 'default target has been set)')
                    else:
                        _inf('(targetless)')
        continue

    # Make the targets in a target-group run in order
    if TARGET and TARGET in TARGET_GROUPS:
        goals = TARGET_GROUPS[TARGET]
    elif TARGET:
        goals = [TARGET]
    else:
        goals = [None]
    for goal in goals:
        for K in yd:
            if K[:5] == 'steps':
                try:
                    tgt = varsub(K[6:])
                except ValueError as ex:
                    _err(str(ex))
                if goal and tgt != goal:
                    _dbg('Skipping %s (target: %s, goal: %s)' \
                         % (tgt, TARGET, goal))
                    continue
                _dbg('Examining step list %s' % K)
                STEPNO = 0
                while STEPNO < len(yd[K]):
                    STEPNO += 1
                    step = yd[K][STEPNO - 1] # STEPNO is 1-based for printing
                    # We're grabbing the first key in the dictionary.
                    # This works only because there is always only one key.
                    step_key = '(none)'
                    for step_key in step:
                        break
                    LINENO = '(%s)' % step_key
                    try:
                        rsrc_pathname = varsub(step_key)
                    except ValueError as ex:
                        _err(str(ex))
                    if rsrc_pathname == 'end-loop':
                        if not LOOP_STACK:
                            _err('Found "end-loop" outside of any loop.')
                        if end_null_loop():
                            continue
                        if increment_loop_index():
                            STEPNO = LOOP_STACK[-1][0]
                        continue
                    if rsrc_pathname == 'loop':
                        if check_null_loop():
                            add_null_loop()
                            continue
                        start_loop(STEPNO, step[step_key])
                        continue
                    if check_null_loop():
                        _dbg('Inside a null loop. Skipping to the end.')
                        continue # If we're inside a null loop, skip to the end-loop
                    if rsrc_pathname == 'print':
                        if isinstance(step[step_key], str):
                            try:
                                sys.stdout.write('%s\n' % \
                                 varsub(step[step_key]))
                            except ValueError as ex:
                                _err(str(ex))
                        else:
                            _wrn('%s %s' \
                             % ('Print statements can only use string',
                                'arguments (with or without variables).'))
                        continue
                    if rsrc_pathname == 'clear-all-output':
                        if step[step_key]:
                            _inf('Running %s' % rsrc_pathname)
                            clear_all_output()
                        continue
                    if rsrc_pathname == 'clear-one-output':
                        if step[step_key]:
                            _inf('Running %s' % rsrc_pathname)
                            clear_one_output(varsub(step[step_key]))
                        continue
                    if rsrc_pathname == 'region':
                        set_region(step[step_key])
                        continue
                    if rsrc_pathname == 'profile':
                        set_profile(step[step_key])
                        continue
                    if rsrc_pathname[:4] == 'var.':
                        ky = rsrc_pathname[4:]
                        rawval = step[step_key]
                        set_var(ky, rawval)
                        continue
                    if os.path.exists('%s/%s' \
                                     % (OUTDIR, rsrc_pathname)):
                        _dbg('%s has already been run.' % rsrc_pathname)
                        continue
                    if rsrc_pathname[0] == '+':
                        rsrc_pathname = rsrc_pathname[1:]
                    WORDS = rsrc_pathname.split('.')
                    if WORDS[0] == 'aws':
                        CMDTUPLE = (WORDS[1], WORDS[2])
                        argtupleraw = step[step_key]
                        _inf('Running %s' % rsrc_pathname)
                        OUTDICT = aws_call(
                            REGION, PROFILE, CMDTUPLE,
                            varsub_tuple(argtupleraw))
                        store_output(rsrc_pathname, OUTDICT)
                    elif WORDS[0] == 'exec':
                        CMD = WORDS[1]
                        argtupleraw = step[step_key]
                        _inf('Running %s' % rsrc_pathname)
                        OUTDICT = exec_call(
                            CMD, varsub_tuple(argtupleraw))
                        store_output(rsrc_pathname, OUTDICT)
                    elif WORDS[0] == 'ssh':
                        argtupleraw = step[step_key]
                        _inf('Running %s' % rsrc_pathname)
                        ARGS = ['ssh']
                        ARGS.extend(varsub_tuple(argtupleraw))
                        response = subprocess.call(ARGS)
                    else:
                        _err('Unrecognized provider: %s' % WORDS[0])
