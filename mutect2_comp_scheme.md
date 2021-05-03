# Mutect2 Comparison scheme:

First 3 sections demonstrate how each mode would be written individually, the last 2 sections are proposals on how all 3 modes should be written into a single joint module.

## PON Genertion:

Simplest set of required inputs, germline resource isn't required, but is used by nibsc build pon pipeline. I think it would still be easier to pass it in via options.args if users wish?

    input:
    tuple val(meta), path(bam), path(bai)
    path fasta
    path fastaidx

    output:
    tuple val(meta), path("*.vcf.gz"), emit: vcf    
    path  '*.version.txt'         , emit: version

    script:
    def software = getSoftwareName(task.process)
    def prefix   = options.suffix ? "${meta.id}${options.suffix}" : "${meta.id}"

    """

    gatk Mutect2 \\
        -R $fasta \\
        -I $bam \\
        -O ${prefix}.vcf.gz \\
        $options.args

    gatk --version | grep Picard | sed "s/Picard Version: //g" > ${software}.version.txt    

    """

## Single Sample Calling:

Germline resource and panel of normals are required for this.

    input:
    tuple val(meta), path(bam), path(bai)
    path fasta
    path fastaidx
    path germline_resource
    path panel_of_normals

    output:
    tuple val(meta), path("*.vcf.gz"), emit: vcf    
    path  '*.version.txt'         , emit: version

    script:
    def software = getSoftwareName(task.process)
    def prefix   = options.suffix ? "${meta.id}${options.suffix}" : "${meta.id}"

    """

    gatk Mutect2 \\
        -R $fasta \\
        -I $bam \\
        --germline-resource $germline_resource \\
        --panel-of-normals $panel_of_normals \\
        -O ${prefix}.vcf.gz \\
        $options.args

    gatk --version | grep Picard | sed "s/Picard Version: //g" > ${software}.version.txt

    """

## TN-Calling:

I think this should be the "default" state for the module to run in as, at least for our pipelines, TN calling is the most frequently used. germline resource and panel of normals both required.

Both tumor and normal bam files are used for this, mutect2 needs to know which is which for this (using the -normal argument). How best to deal with this is discussed in the two joint proposals.

    input:
    tuple val(meta), path(norm_bam), path(norm_bai), path(tumor_bam), path(tumor_bai)
    path fasta
    path fastaidx
    path germline_resource
    path panel_of_normals

    output:
    tuple val(meta), path("*.vcf.gz"), emit: vcf    
    path  '*.version.txt'         , emit: version
    tuple val(meta), path('*tar.gz'), optional:true, emit: dummy.tar.gz

    script:
    def software = getSoftwareName(task.process)
    def prefix   = options.suffix ? "${meta.id}${options.suffix}" : "${meta.id}"

    """

    gatk Mutect2 \\
        -R $fasta \\
        -I $tumor_bam \\
        -I $norm_bam \\
        -normal $norm_bam \\
        --germline-resource $germline_resource \\
        --panel-of-normals $panel_of_normals \\
        --f1r2-tar-gz \\
        -O ${prefix}.vcf.gz \\
        $options.args

    gatk --version | grep Picard | sed "s/Picard Version: //g" > ${software}.version.txt

    """

## Joint Proposal 1, pass tumor/normal in separately:

### mutect2_args val:
a collection of true/false values, same format as meta used to control what mode the module runs in. Used this structure, rather than separate vals as I thought it would be best to keep them all grouped together.

for this proposal mutect2 contains:

- mutect2_args.build_pon:     when true, arguments needed for build pon are used
- mutect2_args.single_call    when true, arguments for single calling
- mutect2_args.single_tumor   when true, single call will use, this is specific to this proposal

Since germline resource is not needed for pon, it is not listed, but if its thought best to, could add a mutect2_arg similar to single_tumor.

### bam inputs:
In this proposal the tumor and normal bams are passed in separately. This is simpler for tumor/normal calling and might make it easier to handle multiple samples if we added this later. However, it means needing a setting for which set of bams single calling uses and less conventional than how bams are handled in other modules.

### code:
    input:
    tuple val(meta), path(norm_bam), path(norm_bai), path(tumor_bam), path(tumor_bai)
    val(mutect2_args)
    path fasta
    path fastaidx
    path germline_resource
    path panel_of_normals

    output:
    tuple val(meta), path("*.vcf.gz"), emit: vcf    
    path  '*.version.txt'         , emit: version
    tuple val(meta), path('*tar.gz'), optional:true, emit: dummy.tar.gz

    script:
    def software = getSoftwareName(task.process)
    def prefix   = options.suffix ? "${meta.id}${options.suffix}" : "${meta.id}"

    if(mutect2_args.build_pon) {
      def $inputsCommand = "-I $norm_bam"
      def $panelsCommand = ''

    } else if(mutect2_args.single_call) {

      if(mutect2_args.single_normal) {
        def $inputsCommand = "-I $normal_bam"

      } else {

        def $inputsCommand = "-I $tumor_bam"
      }

      def $panelsCommand = "--germline-resource $germline_resource \\
      --panel-of-normals $panel_of_normals"

    } else {

      def $inputsCommand = "-I $tumor_bam \\
      -I $normal_bam \\
      -normal $normal_bam"

      def $panelsCommand = "--germline-resource $germline_resource \\
      --panel-of-normals $panel_of_normals \\
      --f1r2-tar-gz"
    }

    """

    gatk Mutect2 \\
        -R $fasta \\
        $inputsCommand \\
        $panelsCommand \\
        -O ${prefix}.vcf.gz \\
        $options.args

        gatk --version | grep Picard | sed "s/Picard Version: //g" > ${software}.version.txt    

    """

## Joint Proposal 2, pass all bams in together

### mutect2_args val:

for this proposal mutect2 contains:

- mutect2_args.build_pon:     when true, arguments needed for build pon are used
- mutect2_args.single_call    when true, arguments for single calling

Since bams is used, the single calls mode doesn't need a setting in mutect2_args for this proposal.

### bam inputs:
in this proposal the tumor and normal bams are passed in via bams and a separate path is used to pass the normal bam to -normal. The passing of the same path twice seems a bit redundant, but it keeps everything else more conventional by using bams and still allowing us to set -normal relatively simply.

### code:

    input:
    tuple val(meta), path(bam), path(bai), path(which_norm)
    val(mutect2_args)
    path fasta
    path fastaidx
    path germline_resource
    path panel_of_normals

    output:
    tuple val(meta), path("*.vcf.gz"), emit: vcf    
    path  '*.version.txt'         , emit: version
    tuple val(meta), path('*tar.gz'), optional:true, emit: dummy.tar.gz

    script:
    def software = getSoftwareName(task.process)
    def prefix   = options.suffix ? "${meta.id}${options.suffix}" : "${meta.id}"

    if(mutect2_args.build_pon) {
      def $inputsCommand = "-I $bam"
      def $panelsCommand = ''

    } else if(mutect2_args.single_call) {

      def $inputsCommand = "-I $bam"      
      def $panelsCommand = "--germline-resource $germline_resource \\
      --panel-of-normals $panel_of_normals"

    } else {

      def $inputsCommand = "-I bam[0] \\
      -I $bam[1] \\
      -normal $which_norm"

      def $panelsCommand = "--germline-resource $germline_resource \\
      --panel-of-normals $panel_of_normals \\
      --f1r2-tar-gz"
    }

    """

    gatk Mutect2 \\
        -R $fasta \\
        $inputsCommand \\
        $panelsCommand \\
        -O ${prefix}.vcf.gz \\
        $options.args

        gatk --version | grep Picard | sed "s/Picard Version: //g" > ${software}.version.txt    

    """

## Other modes:

There are a few other modes for mutect2, in order to keep the proposal less clutered. I wanted to decide which proposal to use first, as the following should be relatively easier to update the module to include as they only require a few setting changes or multiple samples to be passed at the same time, if they are of interest to others.

- joint tumor can call multiple samples at once.
- mitochondrial mode
- force-calling mode
