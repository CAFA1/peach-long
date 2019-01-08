<!-- TOC -->
- [1. 命令](#1-命令)
- [2. main](#2-main)
    - [2.1 peachrun = Engine() //liu](#21-peachrun--engine-liu)
    - [2.2 args.parser = PitXmlAnalyzer() //liu](#22-argsparser--pitxmlanalyzer-liu)
    - [2.3 peach = args.parser.asParser(args.pit)//liu](#23-peach--argsparserasparserargspitliu)
    - [2.5 save_peach_pid(agent=False) //liu](#25-save_peach_pidagentfalse-liu)
    - [2.6 peachrun.Run(args) //liu engine.py](#26-peachrunrunargs-liu-enginepy)
        - [2.6.1 self.peach = args.parser.asParser(args.pit)](#261-selfpeach--argsparserasparserargspit)
        - [2.6.2 self._runPathTest(run, test) #liu](#262-self_runpathtestrun-test-liu)
- [3 self._runTest(run, test, False, self.testRange)](#3-self_runtestrun-test-false-selftestrange)
    - [3.1 actionValues = stateEngine.run(mutator) #liu 开始测试 状态机](#31-actionvalues--stateenginerunmutator-liu-开始测试-状态机)
        - [3.1.1 self._runState(newState, mutator) #liu 执行这个state  state.py](#311-self_runstatenewstate-mutator-liu-执行这个state--statepy)
            - [3.1.1.1 self._runAction(action, mutator) #liu 执行action](#3111-self_runactionaction-mutator-liu-执行action)
                - [3.1.1.1.1  mutator.onDataModelGetValue(action, action.template) #liu](#31111--mutatorondatamodelgetvalueaction-actiontemplate-liu)
                - [3.1.1.1.2 action.value = action.template.getValue()](#31112-actionvalue--actiontemplategetvalue)
    - [!! 3.2 mutator.next() #liu 选择下一个要变异的字段 sequential.py](#-32-mutatornext-liu-选择下一个要变异的字段-sequentialpy)
- [4. action.value = action.template.getValue() #liu 在dom.py :: template class 获取的是变异以后的值](#4-actionvalue--actiontemplategetvalue-liu-在dompy--template-class-获取的是变异以后的值)
    - [4.1 getValue #liu class DataElement(Mutatable)](#41-getvalue-liu-class-dataelementmutatable)
        - [4.1.1 value = self.getRawValue(sout) #liu 获取值](#411-value--selfgetrawvaluesout-liu-获取值)
        - [4.1.2 return self.getInternalValue(sout)](#412-return-selfgetinternalvaluesout)
- [!! 5. mutator.onDataModelGetValue(action, action.template) #liu 变异在这里](#-5-mutatorondatamodelgetvalueaction-actiontemplate-liu-变异在这里)
    - [5.1 self.mutator.sequentialMutation(node) #liu 持续变异](#51-selfmutatorsequentialmutationnode-liu-持续变异)
        - [5.1.1 data = self._performMutation(data, count, rand) #liu 迭代变异多次](#511-data--self_performmutationdata-count-rand-liu-迭代变异多次)
- [6. parse dom 结构](#6-parse-dom-结构)
- [6.1 self.peach = args.parser.asParser(args.pit)](#61-selfpeach--argsparserasparserargspit)
- [6.2 return ParseTemplate(self.configs).parse(uri)](#62-return-parsetemplateselfconfigsparseuri)
- [6.3 return self.parseString(xmlString, findConfigs=False)](#63-return-selfparsestringxmlstring-findconfigsfalse)
- [6.4 return self.HandleDocument(doc, findConfigs=findConfigs)](#64-return-selfhandledocumentdoc-findconfigsfindconfigs)
- [6.5 HandleDocument](#65-handledocument)
- [6.6 HandleTemplate](#66-handletemplate)
- [6.7 HandleDataContainerChildren](#67-handledatacontainerchildren)
- [6.8 self._HandleDataContainerChildren(node, child, parent, errorOnUnknown) #liu](#68-self_handledatacontainerchildrennode-child-parent-erroronunknown-liu)
- [6.9 HandleString](#69-handlestring)
- [6.10](#610)
<!-- /TOC -->
# 1. 命令
```bash
./peach.py -pit Pits/hello.xml -1 -debug
```
# 2. main
```python
if __name__ == '__main__':
    parser = argparse.ArgumentParser(description='Peach Runtime')
    parser.add_argument('-pit', metavar='path', help='pit file') #liu 
    parser.add_argument('-run', metavar='name', help='run name')
    parser.add_argument('-analyzer', nargs="+", help='load analyzer.')
    parser.add_argument('-parser', help='use specific parser.')
    parser.add_argument('-target', help='select a target pit.')
    parser.add_argument('-macros', nargs='+', default=tuple(), help='override configuration macros')
    parser.add_argument('-seed', metavar='#', default=time.time(), help='seed')
    parser.add_argument('-debug', action='store_true', help='turn on debugging. (default: %(default)s)') #liu 
    parser.add_argument('-new', action='store_true', help='use new relations.')
    parser.add_argument('-1', dest='single', action='store_true', help='run single test case.')
    parser.add_argument('-range', nargs=2, type=int, metavar='#', help='run range of test cases.')
    parser.add_argument('-test', action='store_true', help='validate pit file.')
    parser.add_argument('-count', action='store_true', help='count test cases for deterministic strategies.')
    parser.add_argument('-skipto', metavar='#', type=int, help='skip to a test case number.')
    parser.add_argument('-parallel', nargs=2, metavar=('#', '#'), help='use parallelism.')
    parser.add_argument('-agent', nargs=2, metavar=('#', '#'), help='start agent.')
    parser.add_argument('-logging', metavar='#', default=20, type=int, choices=range(10, 60, 10),
                        help='verbosity level of logging')
    parser.add_argument('-check', nargs=2, metavar=('model', 'samples'),
                        help='validate a data model against a set of samples.')
    parser.add_argument('-verbose', action='store_true',
                        help='turn verbosity on. (default: %(default)s)') # Use -vvv action=count
    parser.add_argument('-clean', action='store_true', help='remove python object files.')
    parser.add_argument('-version', action='version', version='%(prog)s 1.0')
    args = parser.parse_args()

    logging.basicConfig(format='[Peach.%(name)s] %(message)s', level=args.logging)
    #print args.pit
    if args.pit and not args.pit.startswith('file:'):
        args.pit = 'file:' + args.pit
    if args.target and not args.target.startswith('file:'):
        args.target = 'file:' + args.target
    #print '2 '+args.pit
    args.configs = {}
    for mac in args.macros:
        k, v = mac.split('=', 1)
        args.configs[k.strip()] = v.strip()
    args.configs['_target'] = args.pit
    #args.pit = args.target

    args.watcher = None
    args.restartFile = None
    peachrun = Engine() //liu 
    peachrun.configs = args.configs
    peachrun.SEED = args.seed
    random.seed(peachrun.SEED)

    if args.debug:
        Engine.debug = True

    if args.clean:
        if sys.platform == "darwin" or sys.platform == "linux2":
            subprocess.call(["find", ".", "-name", ".DS_Store", "-delete"])
            subprocess.call(["find", ".", "-name", "*.pyc", "-delete"])
        elif sys.platform == "win32":
            subprocess.call(["del", "/S", "*.pyc"])
        sys.exit(0)

    if args.analyzer:
        try:
            cls = eval("%s()" % args.analyzer[0])
        except Exception as e:
            fatal("Loading analyzer failed: {}".format(e))
        if hasattr(cls, "supportCommandLine"):
            logging.info("Using %s as analyzer class." % args.analyzer[0])
            a = {}
            for pair in args.analyzer[1:]:
                key, val = pair.split("=")
                a[key] = val
            try:
                cls.asCommandLine(a)
            except Exception as e:
                fatal(e)
        else:
            fatal("Analyzer does not support command line usage.")
        sys.exit(0)

    if args.parser:
        try:
            cls = eval(args.parser)
        except Exception as e:
            fatal("Loading parser class failed: {}".format(e))
        if hasattr(cls, "supportParser"):
            logging.info("Using {} as parser.".format(args.parser))
            args.parser = cls()
        else:
            fatal("Analyzer does not support parser usage.")
    else:
        args.parser = PitXmlAnalyzer() //liu
    args.parser.configs = args.configs

    if args.new:
        Engine.relationsNew = True

    if args.check and args.pit:
        from Peach.Engine.incoming import DataCracker
        dataModelName = args.check[0]
        samplesPath = args.check[1]
        samples = []
        if os.path.isdir(samplesPath):
            for fp in os.listdir(samplesPath):
                samples.append(os.path.join(samplesPath, fp))
        else:
            samples = glob.glob(samplesPath)
        peach = args.parser.asParser(args.pit)//liu 
        dataModel = peach.templates[dataModelName]
        for sample in samples:
            dataModel = peach.templates[dataModelName].copy(peach)
            with open(sample, "rb") as fd:
                data = fd.read()
            buff = PublisherBuffer(None, data, True)
            cracker = DataCracker(peach)
            cracker.optmizeModelForCracking(dataModel, True)
            cracker.crackData(dataModel, buff)
            if dataModel.getValue() == data:
                result = highlight.ok("passed")
            else:
                result = highlight.error("failed")
            logging.info("[%s] cracking: '%s'" % (result, sample))
        logging.info("Done.")
        sys.exit(0)

    if args.single:
        logging.info("Performing a single iteration.")
        Engine.justOne = True

    if args.range:
        if args.range[0] < 0:
            fatal("Count for start must be positive.")
        if args.range[0] >= args.range[1]:
            fatal("Range must be 1 or larger.")
        logging.info("Performing tests from {} -> {}".format(args.range[0], args.range[1]))
        Engine.testRange = args.range

    if args.parallel:
        if args.parallel[0] < 1:
            fatal("Machine count must be >= 2.")
        if args.parallel[0] <= args.parallel[1]:
            fatal("The total number of machines must be less than current machine.")
        logging.debug("Parallel total machines: {}".format(args.parallel[0]))
        logging.debug("Parallel our machine   : {}".format(args.parallel[1]))
    #print args.pit
    if not args.pit and not args.agent:
        logging.error("You must provide a pit or an agent.")
        sys.exit(-1)
    #print '3 '+args.pit
    print args.parser
    if args.test:
        try:
            args.parser.asParser(args.pit)
            logging.debug(highlight.ok("File parsed without errors."))
        except PeachException as e:
            logging.exception(e.msg)
        except etree.LxmlError as e:
            logging.exception("An error occurred while parsing the XML file: {}".format(e))
        except:
            raise
        sys.exit(0)

    if args.count:
        try:
            peachrun.Count(args.parser.asParser(args.pit), args.run)
        except PeachException as e:
            logging.error("Counting test cases only works with deterministic strategies.")
            fatal(e)
        sys.exit(0)

    if args.agent:
        save_peach_pid(agent=True)
        try:
            port = int(args.agent[0])
        except ValueError as e:
            fatal("Agent port is not a valid number.")
        password = args.agent[1]
        try:
            logging.info("Attempting to start Agent ...")
            agent = Agent(password, port)
        except twisted.internet.error.CannotListenError as e:
            fatal(e)
        sys.exit(0)
    else:
        save_peach_pid(agent=False) #liu 

    logging.info("Using random seed: %s" % peachrun.SEED)
    try:
        print "peach run"
        peachrun.Run(args) #liu
    except PeachException as e:
        logging.exception(e.msg)
    except etree.LxmlError as e:
        logging.exception("An error occurred while parsing the XML file: {}".format(e))
    except:
        raise
    finally:
        if DomBackgroundCopier.copyThread is not None:
            DomBackgroundCopier.stop.set()
            DomBackgroundCopier.needcopies.set()
            DomBackgroundCopier.copyThread.join()
            DomBackgroundCopier.copyThread = None
```
## 2.1 peachrun = Engine() //liu 
```python

```
## 2.2 args.parser = PitXmlAnalyzer() //liu
```python

```
## 2.3 peach = args.parser.asParser(args.pit)//liu 
```python

```
## 2.5 save_peach_pid(agent=False) //liu 
```python

```
## 2.6 peachrun.Run(args) //liu engine.py
```python
def Run(self, args):
    runName = "DefaultRun" if args.run is None else args.run
    #print "runname "+runName #liu
    self.restartFile = args.restartFile
    self.restartState = None
    self.verbose = args.verbose
    Engine.verbose = args.verbose
    if args.pit.find(":") >= 0:
        self.pitFile = args.pit[args.pit.find(":") + 1:]
    else:
        self.pitFile = args.pit
    if self.pitFile.find("/") >= 0:
        self.pitFile = os.path.basename(self.pitFile)
    self.peach = args.parser.asParser(args.pit) #liu 解析xml
    run = None
    self.agent = AgentPlexer()
    self._agents = {}
    self.startNum = args.skipto
    self.watcher = EngineWatchPlexer()
    if args.watcher is None:
        self.watcher.watchers.append(StdoutWatcher()) #liu
    else:
        self.watcher.watchers.append(args.watcher)
    if hasattr(self.peach.runs, runName): #liu 查找xml dom树
        run = getattr(self.peach.runs, runName)
    else:
        raise PeachException("Can not find run with name %s." % runName)
    loggers = run.getLoggers()
    if loggers is not None:
        for logger in loggers:
            self.watcher.watchers.append(logger)
    try:
        self.watcher.OnRunStarting(run)  #liu
    except TypeError as t:
        print(t)
        print(dir(self.watcher))
        print(dir(self.watcher.OnRunStarting))
        raise t
    skipToTest = False
    if self.restartFile is not None:
        logging.info("[Restarting] Loading state file %s." % self.restartFile)
        with open(self.restartFile, "rb+") as fd:
            self.restartState = pickle.loads(fd.read())
        skipToTest = True
        skipToTestName = self.restartState[0]
    if args.parallel is None:
        for test in run.tests:
            self._runPathTest(run, test) #liu
        for test in run.tests:
            if skipToTest and test.name != skipToTestName:
                continue
            elif skipToTest and test.name == skipToTestName:
                skipToTest = False
            self._runTest(run, test, False, self.testRange) #liu 
    else:
        logging.info("Configuring run with name %s." % runName)
        if len(run.tests) > 1:
            raise PeachException("Only a single test per-run is currently supported for parallel fuzzing.")
        totalMachines = int(args.parallel[0])
        thisMachine = int(args.parallel[1])
        test = run.tests[0]
        # 1. Get our total count. We want to use a copy of everything so we don't pollute the DOM!
        peach = args.parser.asParser(args.pit)
        totalCount = self._countTest(getattr(peach.runs, runName), getattr(peach.runs, runName).tests[0])
        # 2. How many tests per machine?
        perCount = int(totalCount / totalMachines)
        leftOver = totalCount - (perCount * totalMachines)
        # 3. How many for this machine?
        startCount = thisMachine * perCount
        thisCount = perCount
        if thisMachine == totalMachines - 1:
            thisCount += leftOver
        logging.info("This machine will perform chunk %d through %d out of %d total" %
                (startCount, startCount + thisCount, totalCount))
        self._runTest(run, test, False, [startCount, startCount + thisCount])
    self.watcher.OnRunFinished(run)
```
### 2.6.1 self.peach = args.parser.asParser(args.pit)
```python
#pit.py
class PitXmlAnalyzer(Analyzer):
    """
    Analyzers produce data and state models. Examples of analyzers would be the parsing
    of PeachPit XML files, tokenizing a string, building a data model based on XML file, etc.
    """

    supportParser = True

    def __init__(self):
        self.configs = None

    def asParser(self, uri): #liu 
        """
        Called when Analyzer is used as default Pit parser.
        Should produce a Peach DOM.
        """
        return ParseTemplate(self.configs).parse(uri) #liu 
#parser.py
class ParseTemplate(object):
    """
    The Peach 2 XML -> Peach DOM parser. Uses lxml library.
    Parser returns a top level context object that contains things like templates, namespaces, etc.
    """

    dontCrack = False

    def __init__(self, configs=None):
        self._parser = etree.XMLParser(remove_comments=True) #liu lxml/etree.so
        self._parser.resolvers.add(PeachResolver())
        if configs is None:
            self._configs = {}
        else:
            self._configs = configs

```
### 2.6.2 self._runPathTest(run, test) #liu
```python
    def _runPathTest(self, run, test):
        stateMachine = test.stateMachine
        # If no path declaration found then simply skip the validation
        if not len(stateMachine.getRoute()):
            return
        logging.info("Running path validation test for %s." % test.name)
        #print "liu Running path validation test for %s." % test.name #liu
        try:
            stateEngine = StateEngine(self, stateMachine, test.publishers)
            # Create a path validator to check basic validation rules
            mutator = PathValidationMutator()
            pathValidator = PathValidator(stateEngine.pathFinder, mutator)
            try:
                actionValues = stateEngine.run(mutator)
                print("Traced route: ")
                print(" - ".join(["%s" % str(stateName) for stateName in mutator.states]))
                pathValidator.validate()
            except PathException as e:
                raise PeachException(str(e))
        except PeachException as e:
            logging.error("End of path validation test : Validation failed!")
            raise e
        else:
            logging.info("End of path validation test : Successfully passed")

```
# 3 self._runTest(run, test, False, self.testRange)
```python
    def _runTest(self, run, test, countOnly=False, testRange=None):
        """
        Runs a Test as defined in the Peach XML.

        @type	run: Run object
        @param	run: Run that test is part of
        @type	test: Test object
        @param	test: Test to run
        @type	countOnly: bool
        @param	countOnly: Should we just get total mutator count? Defaults to False.
        @type	testRange: list of numbers
        @param	testRange: Iteration # test ranges.  Only used when performing parallel fuzzing.

        @rtype: number
        @return: the total number of test iterations or None
        """
        stateMachine = test.stateMachine
        stateEngine = StateEngine(self, stateMachine, test.publishers)
        totalTests = "?"
        testCount = 0
        self._startAgents(run, test)
        if not countOnly:
            self.watcher.OnTestStarting(run, test, totalTests)
        for p in test.publishers:
            p.initialize()
        errorCount = 0
        maxErrorCount = 10
        # Get all the mutators we will use
        self.mutators = []
        for m in test.getMutators(): #liu 添加所有的mutator变异器
            try:
                self.mutators.append(eval(m.name))
            except:
                try:
                    self.mutators.append(evalEvent("PeachXml_" + m.name, {}, run))
                except:
                    raise PeachException(
                        "Unable to load mutator [%s], please verify it was imported correctly." % m.name)
        mutator = test.mutator
        value = "StateMachine"
        if self.restartState is not None:
            logging.info("State will load in 1 iteration.")
        elif testRange is not None:
            logging.info("Skipping to start of chunk in 1 iteration.")
        # Needs to be off on its own!
        startCount = None
        endCount = None
        if testRange is not None:
            startCount = testRange[0]
            endCount = testRange[1]
        if self.startNum is not None:
            startCount = self.startNum
        redoCount = 0
        saveState = False
        exitImmediate = False
        actionValues = None
        try:
            while True:
                try:
                    testCount += 1
                    # What if we are just counting?
                    if testCount == 2 and countOnly:
                        self._stopAgents(run, test)
                        return mutator
                    # Go through one iteration before we load state.
                    elif testCount == 2 and self.restartState is not None:
                        logging.info("Restoring state.")
                        testCount = self.restartState[1]
                        mutator.setState(self.restartState[2])
                    elif testCount == 2 and startCount is not None and startCount > 2:
                        # Skip ahead to start range, but not if we are restoring saved state.
                        logging.info("Skipping ahead to iteration %d." % startCount)
                        #testCount -= 1
                        for _ in range(testCount, startCount):
                            mutator.next()
                            testCount += 1
                    # Update total test count
                    if testRange is None:
                        totalTests = mutator.getCount()
                    else:
                        # If we are parallel use our endCount which will also cause the estimated time
                        # left to be correct.
                        totalTests = endCount + 1
                    if totalTests == -1 or totalTests is None:
                        totalTests = "?"
                    else:
                        self.watcher.setTotalVariations(totalTests)
                    # Fire some events
                    self.agent.OnTestStarting()
                    if not countOnly:
                        self.watcher.OnTestCaseStarting(run, test, testCount)
                    self.testCount = testCount
                    mutator.onTestCaseStarting(test, testCount, stateEngine)
                    # Run the test
                    try:
                        actionValues = stateEngine.run(mutator) #liu 开始测试
                    except RedoTestException:
                        raise
                    except MemoryError:
                        # Some tests cause out of memeory exceptions, let skip past them.
                        logging.warning("Out of memory, going to next test.")
                        pass
                    except OverflowError:
                        # Some tests cause out of memeory exceptions, let skip past them.
                        logging.warning("Out of memory, going to next test.")
                        pass
                    except SoftException as e:
                        # In the case of the first iteration we should never fail.
                        if testCount == 1:
                            raise PeachException("Error: First test case failed: ", e)
                        # Otherwise ignore any SoftExceptions and head for next iteration.
                        pass
                    # Pause as needed
                    time.sleep(run.waitTime)
                    mutator.onTestCaseFinished(test, testCount, stateEngine)
                    # Notify
                    if not countOnly:
                        self.watcher.OnTestCaseFinished(run, test, testCount, actionValues)
                    self.agent.OnTestFinished()
                    # Should we repeat this test?
                    if self.agent.RedoTest():
                        logging.warning(highlight.warning("Repeating test"))
                        raise RedoTestException()
                    if self.agent.DetectedFault():
                        logging.warning(highlight.warning("Detected fault! Processing data..."))
                        results = self.agent.GetMonitorData()
                        mutator.onFaultDetected(test, testCount, stateEngine, results, actionValues)
                        self.watcher.OnFault(run, test, testCount, results, actionValues)
                        self.agent.OnFault()
                    # Check for stop event
                    if self.agent.StopRun():
                        logging.warning(highlight.warning("Detected StopRun, bailing!"))
                        self.watcher.OnStopRun(run, test, testCount, None, actionValues)
                        break
                    # Increment our mutator
                    mutator.next() #liu 选择下一个要变异的字段
                    # Reset the redoCounter
                    redoCount = 0
                except RedoTestException as e:
                    if redoCount == 3:
                        raise PeachException(e.message)
                    redoCount += 1
                    testCount -= 1
                except PathException:
                    # Ignore PathException while running tests
                    mutator.next()
                except SoftException:
                    mutator.next()
                # Have we completed our range?
                if (testRange is not None and testCount > endCount) or \
                        (Engine.justOne and startCount is None) or \
                        (Engine.justOne and startCount == testCount):
                    logging.info("Completed iteration range.")
                    break
        except MutatorCompleted:
            pass
        except KeyboardInterrupt:
            logging.warning("User canceled run.")
            saveState = True
            exitImmediate = True
        except PeachException as e:
            if e.msg.find("Unable to reconnect to Agent") > -1:
                results = {
                    "_Bucket": "AgentConnectionFailed"
                }
                self.watcher.OnFault(run, test, testCount, results, actionValues)
            raise
        except:
            # Always save state on exceptions
            saveState = True
            self.watcher.OnTestCaseException(run, test, testCount, None)
            raise
        finally:
            try:
                for publisher in test.publishers:
                    if hasattr(publisher, "hasBeenConnected") and publisher.hasBeenConnected:
                        publisher.close()
                        publisher.hasBeenConnected = False
                    if hasattr(publisher, "hasBeenStarted") and publisher.hasBeenStarted:
                        publisher.stop()
                        publisher.hasBeenStarted = False
                    publisher.finalize()
            except:
                pass
            self._stopAgents(run, test)
        if not countOnly:
            self.watcher.OnTestFinished(run, test)
```
## 3.1 actionValues = stateEngine.run(mutator) #liu 开始测试 状态机
```python
    def run(self, mutator):
        """
        Perform a single run of a StateMachine using the provided
        mutator.
        """

        Debug(1, "StateEngine.run: %s" % self.stateMachine.name)

        for pub in self.publishers:
            pub.hasBeenConnected = False
            pub.hasBeenStarted = False

        self.actionValues = []

        mutator.onStateMachineStarting(self)

        try:
            obj = self._getStateByName(self.stateMachine.initialState) #liu 获取initialstate
            if obj is None:
                raise PeachException("Unable to locate initial state \"%s\"." % self.stateMachine.initialState)

            self._runState(obj, mutator) #liu 执行这个状态

        except StateChangeStateException as e:
            # Hack to stop stack recurtion

            newState = e.state

            while True:
                try:
                    self._runState(newState, mutator) #liu 执行这个state
                    break

                except StateChangeStateException as ee:
                    newState = ee.state

        except SoftException:
            # Soft exceptions are okay except for
            # first iteration

            if Peach.Engine.engine.Engine.context.testCount == 1:
                raise PeachException("Error: First test case failed: ", str(sys.exc_info()[1]))

        finally:
            for pub in self.publishers:
                # At end of state machine make sure publisher is closed
                if pub.hasBeenConnected:
                    pub.close()
                    pub.hasBeenConnected = False

                # At end of state machine make sure publisher is stopped
                if pub.hasBeenStarted:
                    pub.stop()
                    pub.hasBeenStarted = False

        mutator.onStateMachineFinished(self)

        return self.actionValues
```
### 3.1.1 self._runState(newState, mutator) #liu 执行这个state  state.py
```python
    def _runState(self, state, mutator):
        """
        Runs a specific State from a StateMachine.
        """

        Debug(1, "StateEngine._runState: %s" % state.name)
        Engine.context.watcher.OnStateEnter(state)

        # First up we need to copy all the action's templates
        # otherwise values can leak all over the place!

        cracker = DataCracker(self.engine.peach) #liu 注意这个类
        for action in state:
            if not isinstance(action, Action):
                continue

            if action.template is None:
                for c in action:
                    if c.elementType == 'actionparam' or c.elementType == 'actionresult':
                        # Copy template from origional first
                        if not hasattr(c, 'origionalTemplate'):
                            if c.elementType == 'actionresult':
                                cracker.optmizeModelForCracking(c.template, True)
                            c.origionalTemplate = c.template
                            c.origionalTemplate.BuildRelationCache()
                            c.origionalTemplate.resetDataModel()
                            c.origionalTemplate.getValue()
                            #self.domCopier.addDom(c.origionalTemplate)

                        # Make a fresh copy of the template
                        del c[c.template.name]
                        c.template = None
                        ##while c.template == None:
                        ##	c.template = self.domCopier.getCopy(c.origionalTemplate)
                        if c.template is None:
                            c.template = c.origionalTemplate.copy(c)
                            #c.template = c.origionalTemplate.clone()
                        c.append(c.template)

                continue

            # Copy template from origional first
            if not hasattr(action, 'origionalTemplate'):
                if action.type == 'input':
                    cracker.optmizeModelForCracking(action.template, True)

                action.origionalTemplate = action.template
                action.origionalTemplate.BuildRelationCache()
                action.origionalTemplate.resetDataModel()
                action.origionalTemplate.getValue()
                #self.domCopier.addDom(action.origionalTemplate)

            # Make a fresh copy of the template
            del action[action.template.name]
            action.template = None
            ##while action.template == None:
            ##	print "0"
            ##	action.template = self.domCopier.getCopy(action.origionalTemplate)
            action.template = action.origionalTemplate.copy(action)
            action.append(action.template)

        # Next setup a few things
        self.actionValues.append([state.name, 'state'])
        mutator.onStateStarting(self, state)

        # EVENT: onEnter
        if state.onEnter is not None:
            environment = {
                'Peach': self.engine.peach,
                'State': state,
                'StateModel': state.parent,
                'peachPrint': self.f,
                'Mutator': mutator,
                'sleep': time.sleep
            }

            evalEvent(state.onEnter, environment, self.engine.peach)

        stopFuzzing = False
        try:
            # Check if this state is marked with a Stop. If so, end fuzzing
            currentPath = self.pathFinder.current()
            if not currentPath:
                if self.pathFinder.canMove():
                    currentPath = self.pathFinder.next()

            if currentPath:
                stopFuzzing = currentPath.stop

            # Advance to the next path on pathFinder
            nextPath = None
            if self.pathFinder.canMove():
                nextPath = self.pathFinder.next()

            try:
                # Start with first action and continue along
                for action in state:
                    if action.elementType != 'action':
                        continue

                    self._runAction(action, mutator) #liu 执行action

            except SoftException:
                # SoftExceptions are fine

                if Peach.Engine.engine.Engine.context.testCount == 1:
                    raise PeachException("Error: First test case failed: ", str(sys.exc_info()[1]))

            # Pass through the nextState?
            if nextPath:
                raise StateChangeStateException(self._getStateByName(nextPath.stateName))

            # EVENT: onExit
            if state.onExit is not None:
                environment = {
                    'Peach': self.engine.peach,
                    'State': state,
                    'StateModel': state.parent,
                    'peachPrint': self.f,
                    'Mutator': mutator,
                    'sleep': time.sleep
                }

                evalEvent(state.onExit, environment, self.engine.peach)

            mutator.onStateFinished(self, state)
            Engine.context.watcher.OnStateExit(state)

        except StateChangeStateException as e:
            # EVENT: onExit
            if state.onExit is not None:
                environment = {
                    'Peach': self.engine.peach,
                    'State': state,
                    'StateModel': state.parent,
                    'peachPrint': self.f,
                    'Mutator': mutator,
                    'sleep': time.sleep
                }

                evalEvent(state.onExit, environment, self.engine.peach)

            mutator.onStateFinished(self, state)
            Engine.context.watcher.OnStateExit(state)
            newState = mutator.onStateChange(state, e.state)
            if newState is None:
                newState = e.state

            # stop fuzzing next state?
            if not stopFuzzing:
                #self._runState(newState, mutator)
                raise StateChangeStateException(newState)
```
#### 3.1.1.1 self._runAction(action, mutator) #liu 执行action
```python
    def _runAction(self, action, mutator):
        Debug(1, "\nStateEngine._runAction: %s" % action.name)

        mutator.onActionStarting(action.parent, action)

        # If publisher property has been given, use referenced Publisher; otherwise the first one
        if action.publisher is not None:
            pub = self._getPublisherByName(action.publisher)

            if pub is None:
                raise PeachException("Publisher '%s' not found!" % action.publisher)
        else:
            pub = self.publishers[0]

        # EVENT: when
        if action.when is not None:
            environment = {
                'Peach': self.engine.peach,
                'Action': action,
                'State': action.parent,
                'StateModel': action.parent.parent,
                'Mutator': mutator,
                'peachPrint': self.f,
                'sleep': time.sleep,
                'getXml': self.getXml,
                'random': random
            }

            if not evalEvent(action.when, environment, self.engine.peach):
                Debug(1, "Action when failed: " + action.when)
                return
            else:
                Debug(1, "Action when passed: " + action.when)

        Engine.context.watcher.OnActionStart(action)

        # EVENT: onStart
        if action.onStart is not None:
            environment = {
                'Peach': self.engine.peach,
                'Action': action,
                'State': action.parent,
                'StateModel': action.parent.parent,
                'Mutator': mutator,
                'peachPrint': self.f,
                'sleep': time.sleep
            }

            evalEvent(action.onStart, environment, self.engine.peach)

        if action.type == 'input':
            action.value = None

            if not pub.hasBeenStarted:
                pub.start()
                pub.hasBeenStarted = True
            if not pub.hasBeenConnected:
                pub.connect()
                pub.hasBeenConnected = True

            # Make a fresh copy of the template
            action.__delitem__(action.template.name)
            action.template = action.origionalTemplate.copy(action)
            action.append(action.template)

            # Create buffer
            buff = PublisherBuffer(pub)
            self.dirtyXmlCache()

            # Crack data
            cracker = DataCracker(self.engine.peach)
            (rating, _) = cracker.crackData(action.template, buff, "setDefaultValue")

            if rating > 2:
                raise SoftException("Was unble to crack incoming data into %s data model." % action.template.name)

            action.value = action.template.getValue()

        elif action.type == 'output':
            if not pub.hasBeenStarted:
                pub.start()
                pub.hasBeenStarted = True
            if not pub.hasBeenConnected:
                pub.connect()
                pub.hasBeenConnected = True

            # Run mutator
            mutator.onDataModelGetValue(action, action.template) #liu 

            # Get value
            if action.template.modelHasOffsetRelation:
                stringBuffer = StreamBuffer()
                action.template.getValue(stringBuffer)

                stringBuffer.setValue("")
                stringBuffer.seekFromStart(0)
                action.template.getValue(stringBuffer)

                action.value = stringBuffer.getValue()

            else:
                action.value = action.template.getValue() #liu 获取action数据

            Debug(1, "Action output sending %d bytes" % len(action.value))

            if not pub.withNode:
                pub.send(action.value) #liu 发送数据
            else:
                pub.sendWithNode(action.value, action.template)

            # Save the data filename used for later matching
            if action.data is not None and action.data.fileName is not None:
                self.actionValues.append([action.name, 'output', action.value, action.data.fileName])

            else:
                self.actionValues.append([action.name, 'output', action.value])

            obj = Element(action.name, None)
            obj.elementType = 'dom'
            obj.defaultValue = action.value
            action.value = obj

        elif action.type == 'call':
            action.value = None

            actionParams = []

            if not pub.hasBeenStarted:
                pub.start()
                pub.hasBeenStarted = True

            # build up our call
            method = action.method
            if method is None:
                raise PeachException("StateEngine: Action of type \"call\" does not have method name!")

            params = []
            for c in action:
                if c.elementType == 'actionparam':
                    params.append(c)

            argNodes = []
            argValues = []
            for p in params:
                if p.type == 'out' or p.type == 'inout':
                    raise PeachException(
                        "StateEngine: Action of type \"call\" does not yet support out or inout parameters (bug in comtypes)!")

                # Run mutator
                mutator.onDataModelGetValue(action, p.template)

                # Get value
                if p.template.modelHasOffsetRelation:
                    stringBuffer = StreamBuffer()
                    p.template.getValue(stringBuffer)
                    stringBuffer.setValue("")
                    stringBuffer.seekFromStart(0)
                    p.template.getValue(stringBuffer)

                    p.value = stringBuffer.getValue()

                else:
                    p.value = p.template.getValue()

                argValues.append(p.value)
                argNodes.append(p.template)

                actionParams.append([p.name, 'param', p.value])

            if not pub.withNode:
                ret = pub.call(method, argValues)
            else:
                ret = pub.callWithNode(method, argValues, argNodes)

            # look for and set return
            for c in action:
                if c.elementType == 'actionresult':
                    self.dirtyXmlCache()

                    print("RET: %s %s" % (ret, type(ret)))

                    data = None
                    if type(ret) == int:
                        data = struct.pack("i", ret)
                    elif type(ret) == long:
                        data = struct.pack("q", ret)
                    elif type(ret) == str:
                        data = ret

                    if c.template.isPointer:
                        print("Found ctypes pointer...trying to cast...")
                        retCtype = c.template.asCTypeType()
                        retCast = ctypes.cast(ret, retCtype)

                        for i in range(len(retCast.contents._fields_)):
                            (key, value) = retCast.contents._fields_[i]
                            value = eval("retCast.contents.%s" % key)
                            c.template[key].defaultValue = value
                            print("Set [%s=%s]" % (key, value))

                    else:
                        cracker = DataCracker(self.engine.peach)
                        cracker.haveAllData = True
                        (rating, _) = cracker.crackData(c.template, PublisherBuffer(None, data, True))
                        if rating > 2:
                            raise SoftException("Was unble to crack result data into %s data model." % c.template.name)

            self.actionValues.append([action.name, 'call', method, actionParams])

        elif action.type == 'getprop':
            action.value = None

            if not pub.hasBeenStarted:
                pub.start()
                pub.hasBeenStarted = True

            # build up our call
            property = action.property
            if property is None:
                raise Exception("StateEngine._runAction(): getprop type does not have property name!")

            data = pub.property(property)

            self.actionValues.append([action.name, 'getprop', property, data])

            self.dirtyXmlCache()

            cracker = DataCracker(self.engine.peach)
            (rating, _) = cracker.crackData(action.template, PublisherBuffer(None, data))
            if rating > 2:
                raise SoftException("Was unble to crack getprop data into %s data model." % action.template.name)

            # If no exception, it worked

            action.value = action.template.getValue()

            if Peach.Engine.engine.Engine.debug:
                print("*******POST GETPROP***********")
                doc = self.getXml()
                print(etree.tostring(doc, method="html", pretty_print=True))
                print("******************")

        elif action.type == 'setprop':
            action.value = None

            if not pub.hasBeenStarted:
                pub.start()
                pub.hasBeenStarted = True

            # build up our call
            property = action.property
            if property is None:
                raise Exception("StateEngine: setprop type does not have property name!")

            value = None
            valueNode = None
            for c in action:
                if c.elementType == 'actionparam' and c.type == "in":
                    # Run mutator
                    mutator.onDataModelGetValue(action, c.template)

                    # Get value
                    if c.template.modelHasOffsetRelation:
                        stringBuffer = StreamBuffer()
                        c.template.getValue(stringBuffer)
                        stringBuffer.setValue("")
                        stringBuffer.seekFromStart(0)
                        c.template.getValue(stringBuffer)

                        value = c.value = stringBuffer.getValue()

                    else:
                        value = c.value = c.template.getValue()

                    valueNode = c.template
                    break

            if not pub.withNode:
                pub.property(property, value)
            else:
                pub.propertyWithNode(property, value, valueNode)

            self.actionValues.append([action.name, 'setprop', property, value])

        elif action.type == 'changeState':
            action.value = None
            self.actionValues.append([action.name, 'changeState', action.ref])
            mutator.onActionFinished(action.parent, action)
            raise StateChangeStateException(self._getStateByName(action.ref))

        elif action.type == 'slurp':
            action.value = None

            #startTime = time.time()

            doc = self.getXml()
            setNodes = doc.xpath(action.setXpath)
            if len(setNodes) == 0:
                print(etree.tostring(doc, method="html", pretty_print=True))
                raise PeachException("Slurp [%s] setXpath [%s] did not return a node" % (action.name, action.setXpath))

            # Only do this once :)
            valueElement = None
            if action.valueXpath is not None:
                valueNodes = doc.xpath(action.valueXpath)
                if len(valueNodes) == 0:
                    print("Warning: valueXpath did not return a node")
                    raise SoftException("StateEngine._runAction(xpath): valueXpath did not return a node")

                valueNode = valueNodes[0]
                try:
                    valueElement = action.getRoot().getByName(str(valueNode.get("fullName")))

                except:
                    print("valueNode: %s" % valueNode)
                    print("valueNode.nodeName: %s" % split_ns(valueNode.tag)[1])
                    print("valueXpath: %s" % action.valueXpath)
                    print("results: %d" % len(valueNodes))
                    raise PeachException("Slurp AttributeError: [%s]" % str(valueNode.get("fullName")))

            for node in setNodes:
                setElement = action.getRoot().getByName(str(node.get("fullName")))

                if valueElement is not None:
                    Debug(1, "Action-Slurp: 1 Setting %s from %s" % (
                        str(node.get("fullName")),
                        str(valueNode.get("fullName"))
                    ))

                    valueElement = action.getRoot().getByName(str(valueNode.get("fullName")))

                    # Some elements like Block do not have a current or default value
                    if valueElement.currentValue is None and valueElement.defaultValue is None:
                        setElement.currentValue = None
                        setElement.defaultValue = valueElement.getValue()

                    else:
                        setElement.currentValue = valueElement.getValue()
                        setElement.defaultValue = valueElement.defaultValue

                    setElement.value = None

                #print " --- valueElement --- "
                #pub.send(valueElement.getValue())
                #print " --- setElement --- "
                #pub.send(setElement.getValue())
                #print " --------------------"

                else:
                    Debug(1, "Action-Slurp: 2 Setting %s to %s" % (
                        str(node.get("fullName")),
                        repr(action.valueLiteral)
                    ))

                    setElement.defaultValue = action.valueLiteral
                    setElement.currentValue = None
                    setElement.value = None

                    #print " - Total time to slurp data: %.2f" % (time.time() - startTime)

        elif action.type == 'connect':
            if not pub.hasBeenStarted:
                pub.start()
                pub.hasBeenStarted = True

            pub.connect()
            pub.hasBeenConnected = True

        elif action.type == 'accept':
            if not pub.hasBeenStarted:
                pub.start()
                pub.hasBeenStarted = True

            pub.accept()
            pub.hasBeenConnected = True

        elif action.type == 'close':
            if not pub.hasBeenConnected:
                # If we haven't been opened lets ignore
                # this close.
                return

            pub.close()
            pub.hasBeenConnected = False

        elif action.type == 'start':
            pub.start()
            pub.hasBeenStarted = True

        elif action.type == 'stop':
            if pub.hasBeenStarted:
                pub.stop()
                pub.hasBeenStarted = False

        elif action.type == 'wait':
            time.sleep(float(action.valueLiteral))

        else:
            raise Exception("StateEngine._runAction(): Unknown action.type of [%s]" % str(action.type))

        # EVENT: onComplete
        if action.onComplete is not None:
            environment = {
                'Peach': self.engine.peach,
                'Action': action,
                'State': action.parent,
                'Mutator': mutator,
                'StateModel': action.parent.parent,
                'sleep': time.sleep
            }

            evalEvent(action.onComplete, environment, self.engine.peach)

        mutator.onActionFinished(action.parent, action)
        Engine.context.watcher.OnActionComplete(action)
```
##### 3.1.1.1.1  mutator.onDataModelGetValue(action, action.template) #liu 
```python
    def onDataModelGetValue(self, action, dataModel):
        """
        Called before getting a value from a data model

        @type	action: Action
        @param	action: Action we are starting
        @type	dataModel: Template
        @param	dataModel: Data model we are using
        """
        # On first test case lets just figure out which data models and fields
        # we will be mutating.
        if self._isFirstTestCase:
            fullName = dataModel.getFullname() #liu 获取datamodel的名称或者路径
            if fullName not in self._dataModels:
                self._dataModels.append(fullName)
                nodes = dataModel.getAllChildDataElements() #liu 获取datamodel的node
                nodes.append(dataModel)
                for node in nodes:
                    if node.isMutable: #liu 是否可以被变异
                        mutators = self._fieldMutators[node.getFullname()] = []
                        # We should also populate the mutators here
                        for m in Engine.context.mutators:
                            if m.supportedDataElement(node):
                                # Need to create new instance from class
                                mutators.append(m(Engine.context, node)) #liu 寻找可以变异这个node的mutator
            return
        # Is this data model we are changing?
        if not self.fieldName.startswith(dataModel.getFullname()):
            return
        node = dataModel.getRoot().getByName(self.fieldName)
        self.mutator.sequentialMutation(node)
```
##### 3.1.1.1.2 action.value = action.template.getValue()
```python

```
## !! 3.2 mutator.next() #liu 选择下一个要变异的字段 sequential.py
```python
    def next(self):
        if len(self._fieldMutators.keys()) == 0:
            raise PeachException("Error: Peach couldn't find any DataModels "
                                 "with elements to fuzz!")
        if self._isFirstTestCase:
            self._isFirstTestCase = False
        if self.mutator is not None:
            try:
                #self._counts[self.mutator] -= 1
                self.mutator.next()
            except MutatorCompleted:
                self._fieldMutators[self.fieldName].remove(self.mutator)
                self.mutator = None
        while len(self._fieldMutators) > 0:
            self.fieldName = self._random.choice(self._fieldMutators.keys()) #liu 随机选择一个字段
            if len(self._fieldMutators[self.fieldName]) == 0:
                del self._fieldMutators[self.fieldName]
                continue
            self.mutator = \
                self._random.choice(self._fieldMutators[self.fieldName]) #liu 随机选择一个变异器
            return
        #for m in self._counts.keys():
        #	print "%s has %d counts left" % (m.name, self._counts[m])
        raise MutatorCompleted()
```

# 4. action.value = action.template.getValue() #liu 在dom.py :: template class 获取的是变异以后的值
```python
    def getValue(self, sout = None):
        """
        Template needs a custom getValue method!
        """
        try:
            # Sometimes a Template becomes a Block
            if self.elementType == 'template':
                self.relationStringBuffer = sout

            return DataElement.getValue(self, sout)

        finally:
            self.relationStringBuffer = None
```
## 4.1 getValue #liu class DataElement(Mutatable)
```python
    def getValue(self, sout = None):
        """
        Get the value of this data element.
        """

        ## Otherwise lets generate and store our value

        # This method can be called while we are in it.
        # so lets not use self.value to hold anything.
        value = None

        if sout is not None:
            sout.storePosition(self.getFullnameInDataModel())

        ## If we have a cached value for ourselves, use it!
        if self.elementType not in ['template', 'block', 'choice', 'flags',
                                    'xmlelement', 'xmlattribute', 'asn1type', 'custom']:
            if self.value is not None and self.finalValue is None\
               and self.currentValue is None and self.fixup is None\
            and not self.hasRelation():

                if sout is not None:
                    sout.write(self.value)

                #print "getValue(%s): Using self.value" % self.name

                return self.value

        if self.transformer is not None:
            #print "getValue(%s): Transformer will be applied" % self.name

            value = self.getRawValue()
            value = self.transformer.transformer.encode(value)

            if sout is not None:
                sout.write(value)

        else:
            #print "getValue(%s): Using getrawvalue" % self.name
            value = self.getRawValue(sout) #liu 获取值
            #print "getValue(%s): Using getrawvalue: %s" % (self.name, type(value))


        # See if we need to repeat ourselvs.
        if not self.isArray():
            count = self.getCount()
            if count > 1:
                #print "getValue(%s): Item is array, %d" % (self.name, count)
                origValue = value
                value *= count

                if sout is not None:
                    sout.write(origValue * (count-1))

        if value is None:
            raise Exception("value is None for %s type %s" % (self.name, self.elementType))

        if self.elementType != 'flag' and type(value) == type(5):
            print("getValue(%s): WHOA, Returning integer!!!" % self.name)
            print("self:", self)
            print("self.name:", self.name)
            print("self.getfullname", self.getFullnameInDataModel())
            print("self.maxOccurs", self.maxOccurs)
            print("self.ref:", self.ref)
            print("self.getInternalValue", self.getInternalValue())
            print("len(self._children)", len(self._children))
            for child in self:
                print("child:", child)
                print("child.name", child.name)
                print("child.getValue", child.getValue())
            raise Exception("WHOA, Returning integer!!")

        self.value = value
        return self.value
```
### 4.1.1 value = self.getRawValue(sout) #liu 获取值
```python
    def getRawValue(self, sout = None):
        """
        Get value for this data element.

        Performs any needed transforms to produce
        value.
        """

        return self.getInternalValue(sout)
    
```
### 4.1.2 return self.getInternalValue(sout)
```python
    def getInternalValue(self, sout = None):
        """
        Return the internal value of this date element.  This
        value comes before any modifications such as packing,
        padding, truncating, etc.

        For Numbers this is the python int value.

        @type	sout: StreamBuffer
        @param	sout: Output stream
        """

        #print "getInternalValue(%s)" % self.name
        value = ""

        # 0. If using a stream store our location
        if sout is not None:
            pos = sout.storePosition(self.getFullnameInDataModel())

        # 1. Override with currentValue

        if self.currentValue is not None:
            #print "getInternalValue(%s): using currentValue" % self.name

            value = self.currentValue
            if sout is not None:
                sout.write(value, self.getFullnameInDataModel())

            return value

        # 2. Get value from children

        for c in self:
            if isinstance(c, DataElement):
                try:
                    if self.fixup is not None or self.transformer is not None:
                        cv = c.getValue()

                        value += cv
                    else:
                        cv = c.getValue(sout) #liu 从子node中读取字符串

                        value += cv

                except:
                    print(sys.exc_info())
                    raise

        # 3. Fixup

        if self.fixup is not None:
            self.fixup.fixup.context = self
            ret = self.fixup.fixup.do_fixup()
            if ret is not None:
                value = ret
                if sout is not None:
                    sout.write(value, self.getFullnameInDataModel())

        return value
```
# !! 5. mutator.onDataModelGetValue(action, action.template) #liu 变异在这里
```python
    def onDataModelGetValue(self, action, dataModel):
        """
        Called before getting a value from a data model

        @type	action: Action
        @param	action: Action we are starting
        @type	dataModel: Template
        @param	dataModel: Data model we are using
        """
        # On first test case lets just figure out which data models and fields
        # we will be mutating.
        if self._isFirstTestCase:  #liu 第一次运行时，需要配置哪些字段需要变异
            fullName = dataModel.getFullname()
            if fullName not in self._dataModels:
                self._dataModels.append(fullName)
                nodes = dataModel.getAllChildDataElements()
                nodes.append(dataModel)
                for node in nodes:
                    if node.isMutable:
                        mutators = self._fieldMutators[node.getFullname()] = []
                        # We should also populate the mutators here
                        for m in Engine.context.mutators:
                            if m.supportedDataElement(node): #liu 是否支持这个字段的变异
                                # Need to create new instance from class
                                mutators.append(m(Engine.context, node))
            return
        # Is this data model we are changing?
        if not self.fieldName.startswith(dataModel.getFullname()):
            return
        node = dataModel.getRoot().getByName(self.fieldName) #liu 选取某个字段
        self.mutator.sequentialMutation(node) #liu 持续变异 
```
## 5.1 self.mutator.sequentialMutation(node) #liu 持续变异 
```python
    def sequentialMutation(self, node):
        # Allow us to skip ahead and always get same number
        rand = random.Random()
        rand.seed(hashlib.sha512(str(self._current)).digest())
        self.changedName = node.getFullnameInDataModel()
        data = node.getInternalValue() #liu 获取内部的原始值
        for i in range(rand.randint(0, 10)):
            if self._len - 1 <= 0:
                count = 0
            else:
                count = rand.randint(0, self._len - 1)
            data = self._performMutation(data, count, rand) #liu 迭代变异多次
        node.currentValue = data #liu 这样node的值就被修改了
```
### 5.1.1 data = self._performMutation(data, count, rand) #liu 迭代变异多次  
```python
    def _performMutation(self, data, position, rand):
        length = len(data)
        if len(data) == 0:
            return data
        # How many bytes to change
        size = rand.choice([1, 2, 4, 8]) #liu 一次变异多少字节
        if (position + size) >= length:
            position = length - size
        if position < 0:
            position = 0
        if size > length:
            size = length
        for i in range(position, position + size):
            byte = struct.unpack('B', data[i])[0]
            byte ^= rand.randint(0, 255) #liu 通过异或运算来变异
            packedup = struct.pack("B", byte)
            data = data[:i] + packedup + data[i + 1:]
        return data

```
# 6. parse dom 结构
```python
#liu 栈回溯
HandleDocument, parser.py:367
parseString, parser.py:184
parse, parser.py:171
asParser, pit.py:25
Run, engine.py:311
<module>, peach.py:246
run, pydevd.py:1068
main, pydevd.py:1658
<module>, pydevd.py:1664
```
# 6.1 self.peach = args.parser.asParser(args.pit)
```python
    def Run(self, args):
        runName = "DefaultRun" if args.run is None else args.run
        #print "runname "+runName #liu
        self.restartFile = args.restartFile
        self.restartState = None
        self.verbose = args.verbose
        Engine.verbose = args.verbose
        if args.pit.find(":") >= 0:
            self.pitFile = args.pit[args.pit.find(":") + 1:]
        else:
            self.pitFile = args.pit
        if self.pitFile.find("/") >= 0:
            self.pitFile = os.path.basename(self.pitFile)
        self.peach = args.parser.asParser(args.pit)
```
# 6.2 return ParseTemplate(self.configs).parse(uri)
```python
    def asParser(self, uri):
        """
        Called when Analyzer is used as default Pit parser.
        Should produce a Peach DOM.
        """
        return ParseTemplate(self.configs).parse(uri)
```
# 6.3 return self.parseString(xmlString, findConfigs=False)
```python
    def parse(self, uri):
        """
        Parse a Peach XML file pointed to by uri.
        """
        logging.info(highlight.info("Parsing %s" % uri))
        doc = etree.parse(uri, parser=self._parser, base_url="http://phed.org").getroot() #liu 直接调用etree库

        if "_target" in self._configs:

            target = etree.parse(self._configs["_target"], parser=self._parser, base_url="http://phed.org").getroot()
            #print "target"
            #print target
            if split_ns(target.tag)[1] != 'Peach':
                raise PeachException("First element in document must be Peach, not '%s'" % target.tag)

            for child in target.iterchildren():
                doc.append(child)

            del self._configs["_target"]

        # try early to find configuration macros
        self.FindConfigurations(doc)

        xmlString = etree.tostring(doc) #liu 转换成字符串
        #print "xmlstring"
        #print xmlString
        return self.parseString(xmlString, findConfigs=False) #liu 
'''
<Peach xmlns="http://phed.org/2008/Peach" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://phed.org/2008/Peach ../peach.xsd" version="1.0" author="Michael Eddington" description="Hello World Example">

        

        
        <Include ns="default" src="file:defaults.xml"/>

        
        <DataModel name="HelloWorldTemplate">
                <String value="Hello World!"/>
        </DataModel>

        <StateModel name="State" initialState="State1">
                <State name="State1">
                        <Action name="long" type="output">
                                <DataModel ref="HelloWorldTemplate"/>
                        </Action>
                </State>
        </StateModel>

        <Test name="HelloWorldTest">
                <StateModel ref="State"/>
                
                <Publisher class="stdout.Stdout"/>
        </Test>

        
        <Run name="DefaultRun" description="Stdout HelloWorld Run">

                <Test ref="HelloWorldTest"/>

        </Run>

<Include ns="default" src="file:defaults.xml"/>

        
        <DataModel name="HelloWorldTemplate">
                <String value="Hello World!"/>
        </DataModel>

        <StateModel name="State" initialState="State1">
                <State name="State1">
                        <Action name="long" type="output">
                                <DataModel ref="HelloWorldTemplate"/>
                        </Action>
                </State>
        </StateModel>

        <Test name="HelloWorldTest">
                <StateModel ref="State"/>
                
                <Publisher class="stdout.Stdout"/>
        </Test>

        
        <Run name="DefaultRun" description="Stdout HelloWorld Run">

                <Test ref="HelloWorldTest"/>

        </Run>

</Peach>
'''
```
# 6.4 return self.HandleDocument(doc, findConfigs=findConfigs)
```python
    def parseString(self, xml, findConfigs=True):
        """
        Parse a string as Peach XML.
        """

        xml = self.substituteConfigVariables(xml)

        doc = etree.fromstring(xml, parser=self._parser, base_url="http://phed.org") #liu 调用etree库再解析一遍
        #xmlString = etree.tostring(doc)
        #print "xmlstring"
        #print xmlString
        return self.HandleDocument(doc, findConfigs=findConfigs)
```
# 6.5 HandleDocument
```python
    def HandleDocument(self, doc, uri="", findConfigs=True):

        if findConfigs and self.FindConfigurations(doc):
            return self.parseString(etree.tostring(doc), findConfigs=False)

        #self.StripComments(doc)
        self.StripText(doc)

        ePeach = doc

        if split_ns(ePeach.tag)[1] != 'Peach':
            raise PeachException("First element in document must be Peach, not '%s'" % ePeach.tag)

        peach = dom.Peach()
        peach.peachPitUri = uri
        #peach.node = doc
        self.context = peach
        peach.mutators = None

        #: List of nodes that need some parse love list of [xmlNode, parent]
        self.unfinishedReferences = []

        for i in ['templates', 'data', 'agents', 'namespaces', 'tests', 'runs']:
            setattr(peach, i, ElementWithChildren())

        # Peach attributes

        for i in ['version', 'author', 'description']:
            setattr(peach, i, self._getAttribute(ePeach, i))

        # The good stuff -- We are going todo multiple passes here to increase the likely hood
        # that things will turn out okay.

        # Pass 1 -- Include, PythonPath, Defaults
        for child in ePeach.iterchildren():
            child_tag = split_ns(child.tag)[1]
            if child_tag == 'Include':
                # Include this file

                nsName = self._getAttribute(child, 'ns')
                nsSrc = self._getAttribute(child, 'src')

                parser = ParseTemplate(self._configs)
                ns = parser.parse(nsSrc)

                ns.name = nsName + ':' + nsSrc
                ns.nsName = nsName
                ns.nsSrc = nsSrc
                ns.elementType = 'namespace'
                ns.toXml = new_instancemethod(dom.Namespace.toXml, ns)

                nss = Namespace()
                nss.ns = ns
                nss.nsName = nsName
                nss.nsSrc = nsSrc
                nss.name = nsName + ":" + nsSrc
                nss.parent = peach
                ns.parent = nss

                peach.append(nss)
                peach.namespaces.append(ns)
                setattr(peach.namespaces, nsName, ns)

            elif child_tag == 'PythonPath':
                # Add a search path

                p = self.HandlePythonPath(child, peach)
                peach.append(p)
                sys.path.append(p.name)

            elif child_tag == 'Defaults':
                self.HandleDefaults(child, peach)

        # one last check for unresolved macros
        for child in ePeach.iterdescendants():
            for k,v in list(child.items()):
                child.set(k, self.substituteConfigVariables(v, final=True))

        # Pass 2 -- Import
        for child in ePeach.iterchildren():
            child_tag = split_ns(child.tag)[1]
            if child_tag == 'Import':
                # Import module

                if child.get('import') is None:
                    raise PeachException("Import element did not have import attribute!")

                importStr = self._getAttribute(child, 'import')

                if child.get('from') is not None:
                    fromStr = self._getAttribute(child, 'from')

                    if importStr == "*":
                        module = __import__(PeachStr(fromStr), globals(), locals(), [PeachStr(importStr)], -1)

                        try:
                            # If we are a module with other modules in us then we have an __all__
                            for item in module.__all__:
                                globals()["PeachXml_" + item] = getattr(module, item)

                        except:
                            # Else we just have some classes in us with no __all__
                            for item in self.GetClassesInModule(module):
                                globals()["PeachXml_" + item] = getattr(module, item)

                    else:
                        module = __import__(PeachStr(fromStr), globals(), locals(), [PeachStr(importStr)], -1)
                        for item in importStr.split(','):
                            item = item.strip()
                            globals()["PeachXml_" + item] = getattr(module, item)

                else:
                    globals()["PeachXml_" + importStr] = __import__(PeachStr(importStr), globals(), locals(), [], -1)

                Holder.globals = globals()
                Holder.locals = locals()

                i = Element()
                i.elementType = 'import'
                i.importStr = self._getAttribute(child, 'import')
                i.fromStr = self._getAttribute(child, 'from')

                peach.append(i)

        # Pass 3 -- Template
        for child in ePeach.iterchildren():
            child_tag = split_ns(child.tag)[1]
            if child_tag == "Python":
                code = self._getAttribute(child, "code")
                if code is not None:
                    exec(code)

            elif child_tag == 'Analyzer':
                self.HandleAnalyzerTopLevel(child, peach)

            elif child_tag == 'DataModel' or child_tag == 'Template':
                # do something
                template = self.HandleTemplate(child, peach) #liu 新建模板
                #template.node = child
                peach.append(template)
                peach.templates.append(template)
                setattr(peach.templates, template.name, template)

        # Pass 4 -- Data, Agent
        for child in ePeach.iterchildren():
            child_tag = split_ns(child.tag)[1]
            if child_tag == 'Data':
                # do data
                data = self.HandleData(child, peach)
                #data.node = child
                peach.append(data)
                peach.data.append(data)
                setattr(peach.data, data.name, data)

            elif child_tag == 'Agent':
                agent = self.HandleAgent(child, None)
                #agent.node = child
                peach.append(agent)
                peach.agents.append(agent)
                setattr(peach.agents, agent.name, agent)

            elif child_tag == 'StateModel' or child_tag == 'StateMachine':
                stateMachine = self.HandleStateMachine(child, peach)
                #stateMachine.node = child
                peach.append(stateMachine)

            elif child_tag == 'Mutators':
                if self._getBooleanAttribute(child, "enabled"):
                    mutators = self.HandleMutators(child, peach)
                    peach.mutators = mutators

        # Pass 5 -- Tests
        for child in ePeach.iterchildren():
            child_tag = split_ns(child.tag)[1]
            if child_tag == 'Test':
                tests = self.HandleTest(child, None)
                #tests.node = child
                peach.append(tests)
                peach.tests.append(tests)
                setattr(peach.tests, tests.name, tests)

            elif child_tag == 'Run':
                run = self.HandleRun(child, None)
                #run.node = child
                peach.append(run)
                peach.runs.append(run)
                setattr(peach.runs, run.name, run)

        # Pass 6 -- Analyzers

        # Simce analyzers can modify the DOM we need to make our list
        # of objects we will look at first!

        objs = []

        for child in peach.getElementsByType(Blob):
            if child.analyzer is not None and child.defaultValue is not None and child not in objs:
                objs.append(child)
        for child in peach.getElementsByType(String):
            if child.analyzer is not None and child.defaultValue is not None and child not in objs:
                objs.append(child)

        for child in objs:
            try:
                analyzer = eval("%s()" % child.analyzer)
            except:
                analyzer = eval("PeachXml_" + "%s()" % child.analyzer)

            analyzer.asDataElement(child, {}, child.defaultValue)

        # We suck, so fix this up
        peach._FixParents()
        peach.verifyDomMap()
        #peach.printDomMap()

        return peach

  
```
# 6.6 HandleTemplate
```python
    def HandleTemplate(self, node, parent):
        """
        Parse an element named Template.  Can handle actual
        Template elements and also reference Template elements.

        e.g.:

        <Template name="Xyz"> ... </Template>

        or

        <Template ref="Xyz" />
        """

        template = None

        # ref

        if node.get('ref') is not None:
            # We have a base template
            obj = self.GetRef(self._getAttribute(node, 'ref'))

            template = obj.copy(parent)
            template.ref = self._getAttribute(node, 'ref')
            template.parent = parent

        else:
            template = Template(self._getAttribute(node, 'name')) #liu 新建模板类
            template.ref = None
            template.parent = parent

        # name

        if node.get('name') is not None:
            template.name = self._getAttribute(node, 'name')

        template.elementType = 'template'

        # mutable

        mutable = self._getAttribute(node, 'mutable')
        if mutable is None or len(mutable) == 0:
            template.isMutable = True

        elif mutable.lower() == 'true':
            template.isMutable = True

        elif mutable.lower() == 'false':
            template.isMutable = False

        else:
            raise PeachException(
                "Attribute 'mutable' has unexpected value [%s], only 'true' and 'false' are supported." % mutable)

        # pointer

        pointer = self._getAttribute(node, 'pointer')
        if pointer is None:
            pass

        elif pointer.lower() == 'true':
            template.isPointer = True

        elif pointer.lower() == 'false':
            template.isPointer = False

        else:
            raise PeachException(
                "Attribute 'pointer' has unexpected value [%s], only 'true' and 'false' are supported." % pointer)

        # pointerDepth

        if node.get("pointerDepth") is not None:
            template.pointerDepth = self._getAttribute(node, 'pointerDepth')

        # children

        self.HandleDataContainerChildren(node, template) #liu 对child操作

        # Switch any references to old name
        if node.get('ref') is not None:
            oldName = self._getAttribute(node, 'ref')
            for relation in template._genRelationsInDataModelFromHere():
                if relation.of == oldName:
                    relation.of = template.name

                elif relation.From == oldName:
                    relation.From = template.name


        #template.printDomMap()
        return template

 
```
# 6.7 HandleDataContainerChildren
```python
    def HandleDataContainerChildren(self, node, parent, errorOnUnknown=True):
        """
        Handle parsing conatiner children.  This method
        will handle children of DataElement types for
        containers like Block, Choice, and Template.

        Can be used by Custom types to create Custom container
        types.

        @type	node: XML Element
        @param	node: Current XML Node being handled
        @type	parent: ElementWithChildren
        @param	parent: Parent of this DataElement
        @type	errorOnUnknown: Boolean
        @param	errorOnUnknonw: Should we throw an error on unexpected child node (default True)
        """
        # children

        for child in node.iterchildren():
            name = self._getAttribute(child, 'name')
            if name is not None and '.' in name:
                # Replace a deep node, can only happen if we
                # have a ref on us.

                if node.get('ref') is None:
                    raise PeachException(
                        "Error, periods (.) are not allowed in element names unless overrideing deep elements when a parent reference (ref). Name: [%s]" % name)

                # Okay, lets locate the real parent.
                obj = parent
                for part in name.split('.')[:-1]:
                    if part not in obj:
                        raise PeachException(
                            "Error, unable to resolve [%s] in deep parent of [%s] override." % (part, name))

                    obj = obj[part]

                if obj is None:
                    raise PeachException("Error, unable to resolve deep parent of [%s] override." % name)

                # Remove periods from name
                child.set('name', name.split('.')[-1])

                # Handle child with new parent.
                self._HandleDataContainerChildren(node, child, obj, errorOnUnknown)

            else:
                self._HandleDataContainerChildren(node, child, parent, errorOnUnknown) #liu
```

# 6.8 self._HandleDataContainerChildren(node, child, parent, errorOnUnknown) #liu
```python
    def _HandleDataContainerChildren(self, node, child, parent, errorOnUnknown=True):
        node_nodeName = split_ns(node.tag)[1]
        child_nodeName = split_ns(child.tag)[1]
        if child_nodeName == 'Block':
            self.HandleBlock(child, parent)
        elif child_nodeName == 'String':
            self.HandleString(child, parent)
        elif child_nodeName == 'Number':
            self.HandleNumber(child, parent)
        elif child_nodeName == 'Flags':
            self.HandleFlags(child, parent)
        elif child_nodeName == 'Flag':
            self.HandleFlag(child, parent)
        elif child_nodeName == 'Blob':
            self.HandleBlob(child, parent)
        elif child_nodeName == 'Choice':
            self.HandleChoice(child, parent)
        elif child_nodeName == 'Transformer':
            parent.transformer = self.HandleTransformer(child, parent)
        elif child_nodeName == 'Relation':
            relation = self.HandleRelation(child, parent)
            parent.relations.append(relation)
        elif child_nodeName == 'Fixup':
            self.HandleFixup(child, parent)
        elif child_nodeName == 'Placement':
            self.HandlePlacement(child, parent)
        elif child_nodeName == 'Hint':
            self.HandleHint(child, parent)
        elif child_nodeName == 'Seek':
            self.HandleSeek(child, parent)
        elif child_nodeName == 'Custom':
            self.HandleCustom(child, parent)
        elif child_nodeName == 'Asn1':
            self.HandleAsn1(child, parent)
        elif child_nodeName == 'XmlElement':
            # special XmlElement reference

            if child.get('ref') is not None:
                # This is our special case, if we ref we suck the children
                # of the ref into our selves.  This is tricky!

                # get and copy our ref
                obj = self.GetRef(self._getAttribute(child, 'ref'), parent.parent)

                newobj = obj.copy(parent)
                newobj.parent = None

                # first verify all children are XmlElement or XmlAttribute
                for subchild in newobj:
                    if not isinstance(subchild, XmlElement) and not isinstance(subchild, XmlAttribute):
                        raise PeachException(
                            "Error, special XmlElement ref case, reference must only have Xml elements!! (%s,%s,%s)" % (
                                subchild.parent.name, subchild.name, subchild))

                # now move over children
                for subchild in newobj:
                    parent.append(subchild)

                # remove replaced element
                if self._getAttribute(child, 'name') in parent:
                    del parent[self._getAttribute(child, 'name')]

            else:
                self.HandleXmlElement(child, parent)

        elif child_nodeName == 'XmlAttribute':
            self.HandleXmlAttribute(child, parent)

        elif errorOnUnknown:
            raise PeachException(
                PeachStr("found unexpected node [%s] in Element: %s" % (child_nodeName, node_nodeName)))
```

# 6.9 HandleString
```python
    def HandleString(self, node, parent):
        # name

        name = self._getAttribute(node, 'name')

        string = String(name, parent)

        # value

        string.defaultValue = self.GetValueFromNodeString(node)
        string.valueType = self._getValueType(node)
        string.defaultValue = self._HandleValueTypeString(string.defaultValue, string.valueType)

        # tokens

        string.tokens = self._getAttribute(node, 'tokens')

        # padCharacter

        if node.get('padCharacter') is not None:
            val = node.get('padCharacter')
            val = val.replace("'", "\\'")
            string.padCharacter = eval("u'''" + val + "'''")

        # type

        if node.get('type') is not None:
            type = self._getAttribute(node, 'type')
            if type is None or len(type) == 0:
                string.type = 'char'

            elif not (type in ['char', 'wchar', 'utf8', 'utf-8', 'utf-16le', 'utf-16be']):
                raise PeachException("Unknown type of String: %s" % type)

            else:
                string.type = type

        # nullTerminated (optional)

        if node.get('nullTerminated') is not None:
            nullTerminated = self._getAttribute(node, 'nullTerminated')
            if nullTerminated is None or len(nullTerminated) == 0:
                nullTerminated = 'false'

            if nullTerminated.lower() == 'true':
                string.nullTerminated = True
            elif nullTerminated.lower() == 'false':
                string.nullTerminated = False
            else:
                raise PeachException("nullTerminated should be true or false")

        # length (bytes)

        if self._getAttribute(node, 'lengthType') == 'calc':
            string.lengthType = self._getAttribute(node, 'lengthType')
            string.lengthCalc = self._getAttribute(node, 'length')
            string.length = -1

        elif node.get('length') is not None:
            length = self._getAttribute(node, 'length')
            if length is None or len(length) == 0:
                length = None

            try:
                if length is not None:
                    string.length = int(length)
                else:
                    string.length = None
            except:
                raise PeachException("length must be a number or missing %s" % length)

        # Analyzer

        string.analyzer = self._getAttribute(node, 'analyzer')

        # common attributes

        self.HandleCommonDataElementAttributes(node, string)


        # Handle any common children

        self.HandleCommonTemplate(node, string)

        parent.append(string)
        return string
```

# 6.10
```python

```