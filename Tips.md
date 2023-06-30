# Please count the coverage within the codes.
## Please call these methods with git diff 

``` java
import org.apache.maven.plugin.logging.Log;
import org.apache.maven.plugin.logging.SystemStreamLog;
import org.jacoco.core.analysis.IBundleCoverage;
import org.jacoco.core.analysis.IClassCoverage;
import org.jacoco.core.analysis.ICounter;
import org.jacoco.core.analysis.ILine;
import org.jacoco.core.analysis.IPackageCoverage;
import org.jacoco.core.analysis.ISourceFileCoverage;
import org.jacoco.core.internal.analysis.CounterImpl;
import org.jacoco.core.analysis.IMethodCoverage;
import org.jacoco.core.analysis.ISourceNode;
import java.util.*;

public class IncrementCodeFilter {
    private static final Log log = new SystemStreamLog();
    public static void filterIncrementCode(IBundleCoverage bundle, Map<String, String> map) {
        log.info("updateCoverage for " + bundle.getName());
        updateCoverage(map, bundle);
    }

    public static Integer countMethod(IBundleCoverage bundle, Map<String, String> map){
        log.info("count method coverage for " + bundle.getName());
        return countChangedMethodCoverage(bundle,map);
    }

    private static void updateCoverage(Map<String, String> map,
                                       IBundleCoverage bundleCoverage) {
        int bundleCoveredLine = 0;
        int bundleMissedLine = 0;
        Collection<IPackageCoverage> packageCoverages = bundleCoverage
                .getPackages();
        Iterator<IPackageCoverage> packageIterator = packageCoverages
                .iterator();
        while (packageIterator.hasNext()) {
            int packageCoveredLine = 0;
            int packageMissedLine = 0;
            IPackageCoverage iPackage = packageIterator.next();
            // update class
            Iterator<IClassCoverage> classIterator = iPackage.getClasses()
                    .iterator();
            while (classIterator.hasNext()) {
                IClassCoverage iClass = classIterator.next();
                String name = iClass.getName().replaceAll("/", ".") + ".java";
                log.info("The name of iClass is " + name);
                if (!map.containsKey(name)) {
                    classIterator.remove();
                    continue;
                }
                String info = map.get(name);
                CounterImpl counter = updateCounterInLine(iClass, info);
                updateMethodCounter(iClass, info);
                iClass.setLineCounter(counter);
            }

            // update source
            Iterator<ISourceFileCoverage> sourceIterator = iPackage
                    .getSourceFiles().iterator();
            while (sourceIterator.hasNext()) {
                ISourceFileCoverage iSource = sourceIterator.next();
                String name = iSource.getPackageName().replaceAll("/", ".")
                        + "." + iSource.getName();

                log.info("The name of iSource is " + name);
                if (!map.containsKey(name)) {
                    sourceIterator.remove();
                    continue;
                }
                String startAndOffset = map.get(name);
                CounterImpl counter = updateCounterInLine(iSource, startAndOffset);
                iSource.setLineCounter(counter);

                packageCoveredLine += counter.getCoveredCount();
                packageMissedLine += counter.getMissedCount();
            }
            // remove empty package
            if (iPackage.getSourceFiles().size() == 0) {
                packageIterator.remove();
            }
            iPackage.setLineCounter(CounterImpl.getInstance(packageMissedLine,
                    packageCoveredLine));
            bundleCoveredLine += packageCoveredLine;
            bundleMissedLine += packageMissedLine;
        }
        bundleCoverage.setLineCounter(
                CounterImpl.getInstance(bundleMissedLine, bundleCoveredLine));
    }

private static void updateMethodCounter(IClassCoverage aClass,
                                            String changeLineNo) {
        for (IMethodCoverage method : aClass.getMethods()) {
            int index = method.getFirstLine();
            int end = method.getLastLine();
            int totalLine = 0;
            int missedLine = 0;
            log.info(" Coverage Method name is " + method.getName());
            String[] changed = changeLineNo.substring(1).split(":");
            log.info("changeLineNo is " + changeLineNo);
            while (index <= end) {
                ILine iLine = method.getLine(index);
                if(!ifLineNumberInChangedLineOffset(changed, index)){
                    method.replaceLine(index, null);
                } else {
                    totalLine++;
                    int s = iLine.getStatus();
                    if (s == ICounter.NOT_COVERED || s == ICounter.PARTLY_COVERED) {
                        missedLine += 1;
                    }
                }
                index++;
            }
            int covered = totalLine - missedLine;
            method.setLineCounter(CounterImpl.getInstance(missedLine, covered));
        }
    }

    private static CounterImpl updateCounterInLine(ISourceNode aSource,
                                                   String changeLineNo) {
        int missedLine = 0;
        int coveredLine = 0;
        int index = aSource.getFirstLine();
        int end = aSource.getLastLine();
        String[] changed = changeLineNo.substring(1).split(":");
        log.info("changeLineNo is " + changeLineNo);
        Arrays.stream(changed).forEach(no -> {
            log.info("Change line after split is " + no);
        });
        while (index <= end) {
            ILine iLine = aSource.getLine(index);
            if(!ifLineNumberInChangedLineOffset(changed, index)){
                aSource.replaceLine(index, null);
            } else {
                int s = iLine.getStatus();
                if (s == ICounter.NOT_COVERED || s == ICounter.PARTLY_COVERED) {
                    missedLine += 1;
                } else if (s == ICounter.EMPTY) {
                    // ignore
                } else {
                    coveredLine += 1;
                }
            }
            index++;
        }
        return CounterImpl.getInstance(missedLine, coveredLine);
    }

    private static Boolean ifLineNumberInChangedLineOffset(String[] changedLine, Integer lineNumber){
        for(String lineAndOffset: changedLine){
            String[] line = lineAndOffset.split(",");
            if(lineNumber >= Integer.valueOf(line[0]) && lineNumber < (Integer.valueOf(line[0]) + Integer.valueOf(line[1]))){
                return true;
            }
        }
        return false;
    }

    private static Integer countChangedMethodCoverage(IBundleCoverage bundleCoverage,Map<String,String> signatureMap) {
        log.info("count changed method coverage...");
        for(Map.Entry<String,String> entry : signatureMap.entrySet()) {
            log.info(" methodMap key is " + entry.getKey() + " ; value is " + entry.getValue());
        }
        Collection<IPackageCoverage> packageCoverages = bundleCoverage
                .getPackages();
        Iterator<IPackageCoverage> packageIterator = packageCoverages
                .iterator();
        Integer changedMethodCoverage = 0;
        while (packageIterator.hasNext()) {
            IPackageCoverage iPackage = packageIterator.next();
            Iterator<IClassCoverage> classIterator = iPackage.getClasses().iterator();
            while (classIterator.hasNext()) {
                IClassCoverage iClass = classIterator.next();
                String name = iClass.getName().replaceAll("/", ".") + ".java";
                log.info("The name of iClass is " + name);
                if (!signatureMap.containsKey(name)) {
                    classIterator.remove();
                    continue;
                }
                changedMethodCoverage += countChangedMethodInIMethod(iClass, signatureMap.get(name));
            }
        }
        return changedMethodCoverage;
    }

    private static Integer countChangedMethodInIMethod(IClassCoverage aClass, String methodName){
        Integer coverage = 0;
        String[] methodArr = methodName.split(":");
        Map<String, Integer> map = new HashMap<String, Integer>();
        for(String name: methodArr){
            if(!map.containsKey(name)){
                map.put(name,1);
            } else {
                map.put(name, map.getOrDefault(name,0) + 1);
            }
        }
        for(IMethodCoverage method : aClass.getMethods()){
            log.info("The method name is " + method.getName());
            int index = method.getFirstLine();
            int end = method.getLastLine();
            if(map.containsKey(method.getName())){
                while(index <= end){
                    ILine iLine = method.getLine(index);
                    int s = iLine.getStatus();
                    //if (s != ICounter.NOT_COVERED && s != ICounter.PARTLY_COVERED) {
                    if(s == ICounter.FULLY_COVERED ){
                        coverage += 1;
                        Integer current = map.getOrDefault(method.getName(),0);
                        if(current <= 1 ){
                            map.remove(method.getName());
                        } else {
                            map.put(method.getName(),current - 1);
                        }
                        break;
                    }
                    index++;
                }
                /*
                log.info("The matched method name is " + method.getName());
                coverage += 1;
                Integer current = map.getOrDefault(method.getName(),0);
                if(current <= 1 ){
                    map.remove(method.getName());
                } else {
                    map.put(method.getName(),current - 1);
                }

                 */
            }
        }
        return coverage;
    }

}

```